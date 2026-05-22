# INC: Tráfego das VMs libvirt sem internet — Docker FORWARD policy DROP

- **Data:** 2026-05-21
- **Local:** PC pessoal
- **Sistema:** Fedora 44, KDE Plasma 6, Docker 29.5.1, libvirt recém-instalado

---

## Contexto / Sintoma

Ao instalar libvirt num host onde o Docker já estava rodando, VMs criadas com a rede default (`virbr0`, `192.168.122.0/24`) recebem IP via DHCP, pingam o gateway `192.168.122.1` (a própria bridge), mas **não conseguem sair para a internet**. Pings para `8.8.8.8` falham, resolução DNS de domínios externos não funciona.

Este incidente já tinha acontecido em uma instalação anterior. Não lembrava dos detalhes do fix, mas lembrava que o Docker era o culpado — por isso desta vez foi tratado preventivamente como parte do setup (ver `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md`).

---

## Causa Raiz

O Docker, ao iniciar, **reescreve a chain `FORWARD` do iptables com policy `DROP`**. Isso é uma proteção do Docker — ele controla explicitamente qual tráfego passa pelas bridges dos containers e dropa tudo que não bate nas regras dele.

O problema: o libvirt cria a `virbr0` esperando que o `FORWARD` tenha policy `ACCEPT` (ou que existam regras explícitas pra `virbr0`). Quando os dois coexistem:

1. VM envia pacote pra `8.8.8.8` via gateway `192.168.122.1`
2. Pacote chega no host, precisa ser forwardado pra interface externa
3. Passa pela chain `FORWARD`, não bate em nenhuma regra ACCEPT do Docker
4. Cai na policy default `DROP`
5. Pacote morre silenciosamente

Sintoma clássico: ping pro gateway funciona (não passa por FORWARD, vai pro INPUT do host), mas ping pra qualquer IP externo falha.

A chain `DOCKER-USER` existe justamente pra isso — é onde regras customizadas devem ser injetadas, e elas sobrevivem aos reinícios do Docker (que limpa as outras chains).

---

## Diagnóstico

```bash
# Confirmar policy do FORWARD
sudo iptables -L FORWARD -n | head -1
# Chain FORWARD (policy DROP 0 packets, 0 bytes)
# ^ Esse "DROP" é a smoking gun

# Ver chain de processamento — DOCKER-USER vem antes das chains do Docker
sudo iptables -L FORWARD -n -v --line-numbers
# Chain FORWARD (policy DROP)
# num   pkts bytes target           ...
# 1     1917  820K ts-forward       (Tailscale)
# 2     1917  820K DOCKER-USER      ← aqui que vamos injetar
# 3     1917  820K DOCKER-FORWARD   (regras do Docker)

# Confirmar que DOCKER-USER está vazia (ou sem regras pra virbr0)
sudo iptables -L DOCKER-USER -n -v --line-numbers
# Chain DOCKER-USER (1 references)
# num   pkts bytes target     ...
# (vazia)

# Confirmar que IP forwarding do kernel está ativo (não é o problema)
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1
```

---

## Solução

Injetar duas regras ACCEPT em `DOCKER-USER`:

```bash
# EXT = interface externa do host (ex: enp4s0, wlp3s0)
EXT=enp4s0

# Tráfego das VMs saindo pra internet
sudo iptables -I DOCKER-USER -i virbr0 -o $EXT -j ACCEPT

# Tráfego de retorno — só de conexões já iniciadas pelas VMs (conntrack)
sudo iptables -I DOCKER-USER -o $EXT -i virbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

**Por que duas regras e não uma com `-j ACCEPT` cego nos dois sentidos:** a regra de retorno com conntrack `RELATED,ESTABLISHED` só libera pacotes que pertencem a conexões iniciadas de dentro pra fora. Isso evita que tráfego entrante novo da internet chegue nas VMs sem autorização explícita.

### Persistir entre reboots

Regras `iptables` somem ao reiniciar. Persistência via systemd unit:

```bash
sudo tee /etc/systemd/system/libvirt-docker-bridge.service > /dev/null <<'EOF'
[Unit]
Description=Allow libvirt virbr0 traffic through Docker FORWARD chain
After=docker.service libvirtd.service
Requires=docker.service libvirtd.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iptables -I DOCKER-USER -i virbr0 -o EXT_PLACEHOLDER -j ACCEPT
ExecStart=/usr/sbin/iptables -I DOCKER-USER -o EXT_PLACEHOLDER -i virbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
ExecStop=/usr/sbin/iptables -D DOCKER-USER -i virbr0 -o EXT_PLACEHOLDER -j ACCEPT
ExecStop=/usr/sbin/iptables -D DOCKER-USER -o EXT_PLACEHOLDER -i virbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

[Install]
WantedBy=multi-user.target
EOF

sudo sed -i "s/EXT_PLACEHOLDER/$EXT/g" /etc/systemd/system/libvirt-docker-bridge.service
sudo systemctl daemon-reload
sudo systemctl enable --now libvirt-docker-bridge.service
```

---

## Verificação

```bash
# Regras presentes
sudo iptables -L DOCKER-USER -n -v --line-numbers
# Esperado: ver as 2 regras ACCEPT

# Service ativo
sudo systemctl status libvirt-docker-bridge.service

# Dentro de uma VM teste — todos devem responder
ping -c 2 192.168.122.1
ping -c 2 8.8.8.8
ping -c 2 google.com
```

---

## Estado final

```
FORWARD policy   : DROP (mantida — é o Docker que define)
DOCKER-USER      : 2 regras ACCEPT pra virbr0 ↔ enp4s0
Persistência     : libvirt-docker-bridge.service habilitado
Boot order       : After docker.service libvirtd.service
VMs              : acesso normal à internet via NAT default do libvirt
```

---

## Como evitar no futuro

- **Em qualquer setup novo onde libvirt e Docker convivem**, aplicar as regras de `DOCKER-USER` como parte do procedimento de instalação, não como reação a um sintoma. Isso já está documentado em `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md`.
- **Se for instalar Docker em um host onde libvirt já roda**, lembrar de aplicar essas regras logo após — porque o Docker vai mudar a policy do FORWARD pra DROP no momento que iniciar.
- **Diagnóstico rápido** quando uma VM nova ficar sem internet: `sudo iptables -L FORWARD -n | head -1` — se a policy for DROP, é provavelmente isso.

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/systemd/system/libvirt-docker-bridge.service` | Criado | Unit que injeta regras ACCEPT em DOCKER-USER no boot |

---

## Referência

- Setup completo: `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md`
- Decisão arquitetural relacionada: `ADR-PC-2026-05-22-libvirt-rede-isolada-do-docker.md`
- Documentação Docker sobre DOCKER-USER: https://docs.docker.com/network/packet-filtering-firewalls/
