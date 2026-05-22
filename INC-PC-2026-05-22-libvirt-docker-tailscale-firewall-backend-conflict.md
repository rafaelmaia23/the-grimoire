# INC: Conflito de backends de firewall — libvirt, Docker, Tailscale e firewalld na tabela ip filter

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, KDE Plasma 6, Docker 29.5.1, libvirt 12.0.0, Tailscale ativo, firewalld nftables backend

---

## Contexto / Sintoma inicial

Após reinstalar libvirt num PC com Docker e Tailscale rodando, VMs criadas com a rede default (`virbr0`, `192.168.122.0/24`) recebem IP via DHCP, pingam o gateway `192.168.122.1`, mas não conseguem sair pra internet. Ping pra `8.8.8.8` falha, TCP pra qualquer IP externo falha, DNS direto pra `1.1.1.1` falha. Único DNS que "funciona" é o que passa pelo Tailscale (resolução interna via AdGuard na VPS).

Esse problema persistiu por horas, com várias tentativas de fix que falharam ou resolveram parcialmente. A causa raiz só foi descoberta após desmontar e remontar todo o stack de rede.

---

## Linha do tempo

### Tentativa 1 — Regra no DOCKER-USER (a "solução" da memória)

Aplicado conforme documentação anterior:

```bash
sudo iptables -I DOCKER-USER -i virbr0 -o enp4s0 -j ACCEPT
sudo iptables -I DOCKER-USER -o enp4s0 -i virbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

Persistência via `libvirt-docker-bridge.service`. **Não resolveu.** VM continuou sem internet. Aqui já ficou claro que a documentação anterior do problema (INC do dia 21) tinha causa raiz errada — o sintoma era similar mas a solução não se aplicava.

### Tentativa 2 — Investigação do firewalld

Descoberto que a zona libvirt do firewalld estava com `masquerade: no` e `forward: no` por padrão no Fedora 44:

```bash
sudo firewall-cmd --zone=libvirt --list-all
# masquerade: no
# forward: no
# rich rules: rule priority="32767" reject
```

Aplicado:
```bash
sudo firewall-cmd --zone=libvirt --add-masquerade --permanent
sudo firewall-cmd --zone=libvirt --add-forward --permanent
sudo firewall-cmd --reload
```

**Não resolveu.** Sintoma mudou de "100% packet loss" pra "Destination Host Unreachable" (mensagem do reject ICMP do firewalld), mas a internet continuou bloqueada. Pior: a regra de `MASQUERADE` esperada não apareceu na chain `nat/POSTROUTING` do iptables.

### Tentativa 3 — Recriação da rede default do libvirt

```bash
sudo virsh net-destroy default
sudo virsh net-start default
```

A rede subiu mas as regras MASQUERADE pra `192.168.122.0/24` continuaram ausentes em `iptables -t nat -L POSTROUTING`. Descoberta importante: as regras **existiam** no `nft list ruleset`, mas em tabela diferente da que `iptables -L` mostrava. Primeira pista de que tinha dois sistemas de firewall coexistindo.

### Tentativa 4 — Policies cross-zone do firewalld

Criadas duas policies pra permitir tráfego entre as zonas:

```bash
sudo firewall-cmd --permanent --new-policy libvirt-to-external
sudo firewall-cmd --permanent --policy libvirt-to-external --add-ingress-zone libvirt
sudo firewall-cmd --permanent --policy libvirt-to-external --add-egress-zone FedoraWorkstation
sudo firewall-cmd --permanent --policy libvirt-to-external --set-target ACCEPT
sudo firewall-cmd --permanent --policy libvirt-to-external --add-masquerade

sudo firewall-cmd --permanent --new-policy external-to-libvirt
# (sentido inverso)
```

**Não resolveu.** Análise do `nft list ruleset` mostrou que as chains das policies estavam vazias (só com `jump pre/log/deny/allow/post`) e o `accept` final não interrompia o fluxo da chain pai como esperado.

### Tentativa 5 — Diagnóstico com tcpdump simultâneo

Em três terminais paralelos, monitorando `vnet0` (interface da VM no host), `virbr0` (bridge libvirt) e `enp4s0` (interface externa). Disparado teste TCP da VM (`bash -c 'cat < /dev/tcp/1.1.1.1/443'`).

**Descoberta crítica:** o pacote SYN saía da VM, aparecia em `vnet0`, aparecia em `virbr0`, mas **não aparecia em `enp4s0`**. Estava sendo dropado entre `virbr0` e `enp4s0` apesar das policies estarem configuradas.

Antes (com regras manuais no DOCKER-USER, sessão anterior): o SYN saía mascarado pela `enp4s0`, mas o SYN-ACK que voltava da Cloudflare era dropado no caminho de retorno (`enp4s0 → virbr0`).

### Tentativa 6 — Hipótese do Tailscale

`sudo systemctl restart tailscaled` e novos testes. **Não resolveu.** Eliminou Tailscale como suspeito direto.

### Tentativa 7 — Remoção do Docker

Removido Docker completamente:
```bash
sudo dnf remove -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo rm -rf /var/lib/docker /etc/docker /var/lib/containerd
```

E chains residuais do Docker no nftables (que sobreviveram ao uninstall do pacote) limpadas manualmente. **Não resolveu.** Refutou definitivamente "Docker é a causa", o que tinha sido a hipótese principal durante toda a investigação.

### Tentativa 8 — Reset estrutural completo

Plano: derrubar tudo (libvirt + firewalld + chains residuais), reinstalar firewalld limpo, configurar libvirt pra usar `nftables` nativo em vez de `iptables-nft`, reinstalar virtualização do zero, e só depois reinstalar Docker.

Executado em fases controladas (ver `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md` pro procedimento completo). **Resolveu.** VM Alpine pegou internet de primeira após o reset.

### Tentativa 9 — Docker reinstalado com nftables nativo

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "firewall-backend": "nftables",
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "dns": ["1.1.1.1", "8.8.8.8"]
}
EOF
```

Docker criou sua própria tabela `ip docker-bridges` no nftables, **sem injetar regras na tabela `ip filter`** compartilhada. VM Alpine continuou com internet. `docker run hello-world` funcionou. Containers e VMs coexistindo sem conflito.

---

## Causa raiz

**Sobreposição de regras na tabela `ip filter` quando múltiplos componentes usam `iptables-nft` (modo padrão de compatibilidade) no mesmo host.**

Cenário no PC antes do reset:

| Componente | Onde criava regras (antes) | Tabela afetada |
|---|---|---|
| Tailscale | iptables-nft (nativo) | `ip filter`, `ip nat` |
| Docker (default `iptables: true`) | iptables-nft | `ip filter`, `ip nat` |
| libvirt (default backend) | iptables-nft | `ip filter`, `ip nat` |
| firewalld (backend nftables) | nftables nativo | `inet firewalld` |

Resultado: **três sistemas escrevendo na mesma tabela `ip filter`**, com prioridades parcialmente sobrepostas, chains chamando jumps em ordens diferentes, e o firewalld adicionando ainda mais uma camada em `inet firewalld` com priority `filter + 10` (rodando depois das chains de `ip filter`).

O efeito prático: pacotes eram processados em ordens imprevisíveis. Às vezes morriam no DROP do FORWARD que o Docker injetava. Às vezes passavam mas eram rejeitados pelo reject do firewalld da zona FedoraWorkstation (que não conhecia a virbr0 como interface legítima de forward). Às vezes ainda saíam mas o retorno não conseguia voltar porque o conntrack tinha sido corrompido por uma das outras chains.

**O sintoma específico do "Docker FORWARD policy DROP"** descrito no INC anterior (deletado) era **um dos muitos efeitos possíveis** dessa sobreposição, não a causa raiz. Por isso a solução daquele INC (regras manuais no DOCKER-USER) funcionou em uma instalação anterior e não funcionou nesta — dependia de qual componente estava controlando a chain `FORWARD` no momento.

### O que faz a solução funcionar

Cada componente em sua **própria tabela nftables**:

| Componente | Backend configurado | Tabela própria |
|---|---|---|
| Tailscale | nativo | `ip filter`, `ip nat` (ainda compartilhadas, mas só Tailscale escreve agora) |
| Docker | `firewall-backend: nftables` | `ip docker-bridges`, `ip6 docker-bridges` |
| libvirt | `firewall_backend = "nftables"` | `ip libvirt_network`, `ip6 libvirt_network` |
| firewalld | nativo | `inet firewalld` |

Sem sobreposição. Cada um processa seus pacotes em sua tabela, com prioridades próprias, sem pisar nos pés dos outros.

---

## Solução final

Reset completo e reinstalação na ordem certa. **Procedimento completo em `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md`.** Resumo:

1. Remover Docker e libvirt completamente
2. `nft delete table` pra todas as tabelas residuais (libvirt_network, docker-bridges, e até as `ip filter`/`ip nat` resto compartilhadas com Tailscale — Tailscale recria ao reiniciar)
3. Reinstalar firewalld (`dnf reinstall`) com config padrão
4. Garantir `FirewallBackend=nftables` no `/etc/firewalld/firewalld.conf`
5. Restart `tailscaled` (recria suas regras em tabelas próprias)
6. Reinstalar `@virtualization`
7. Configurar `firewall_backend = "nftables"` em `/etc/libvirt/network.conf`
8. Subir libvirtd e verificar que criou tabela `libvirt_network` própria com MASQUERADE pra `192.168.122.0/24`
9. Validar com VM teste (Alpine) — deve ter internet imediatamente
10. Reinstalar Docker com `"firewall-backend": "nftables"` em `/etc/docker/daemon.json`
11. Validar que Docker criou tabela `docker-bridges` própria e que a VM continua com internet

---

## Estado final

```
Backend firewall do sistema      : nftables (nativo)
firewalld backend                : nftables (FirewallBackend=nftables)
libvirt firewall_backend         : nftables (em /etc/libvirt/network.conf)
Docker firewall-backend          : nftables (em /etc/docker/daemon.json)
Tailscale                        : iptables-nft (não configurável, mas em chains próprias)

Tabelas nftables presentes:
  - ip filter, ip6 filter         (Tailscale)
  - ip nat, ip6 nat               (Tailscale)
  - ip mangle, ip6 mangle         (sistema)
  - inet firewalld                (firewalld — zonas, policies)
  - ip libvirt_network            (libvirt — NAT 192.168.122.0/24)
  - ip6 libvirt_network           (libvirt IPv6)
  - ip docker-bridges             (Docker — NAT 172.17.0.0/16)
  - ip6 docker-bridges            (Docker IPv6)

Sobreposição entre componentes   : zero
VMs                              : internet OK
Containers                       : internet OK
VMs ↔ Containers                  : isolados (esperado, ver ADR)
```

---

## Como evitar no futuro

### Em qualquer setup novo no Fedora 44+

**Antes de instalar libvirt ou Docker num host com firewalld:**

1. Confirmar que firewalld usa `FirewallBackend=nftables` (default no Fedora 44)
2. Configurar libvirt com `firewall_backend = "nftables"` **antes** de habilitar o `libvirtd`
3. Configurar Docker com `"firewall-backend": "nftables"` no `daemon.json` **antes** do primeiro start

Se ambos rodarem com o padrão (`iptables-nft`), eles vão compartilhar a tabela `ip filter` com qualquer outro processo que use `iptables` ou `iptables-nft` (Tailscale, scripts próprios, etc.), e o conflito é provável.

### Diagnóstico rápido se VM perder internet

```bash
# 1. Lista as tabelas nftables — cada componente deve ter sua própria
sudo nft list tables

# Esperado se setup correto:
# table ip filter            (Tailscale)
# table ip6 filter           (Tailscale)
# table inet firewalld
# table ip libvirt_network   (libvirt, separada)
# table ip docker-bridges    (Docker, separada)

# 2. Se libvirt_network não existe ou está vazia → firewall_backend errado
sudo nft list table ip libvirt_network | head -20
sudo grep firewall_backend /etc/libvirt/network.conf

# 3. Se Docker mexe na tabela ip filter → firewall-backend errado
sudo nft list table ip filter | grep -i docker
sudo cat /etc/docker/daemon.json | grep firewall-backend
```

### Sinais de que o conflito voltou

- VM faz handshake TCP parcial (SYN sai, SYN-ACK aparece em tcpdump na `enp4s0` mas não chega na VM)
- Counters de MASQUERADE pra `192.168.122.0/24` em `nft list table ip libvirt_network` subindo mas a VM continua sem internet
- `iptables -L FORWARD` mostra `policy DROP` em vez de `policy ACCEPT`
- Sintoma "Destination Host Unreachable" da VM em vez de timeout silencioso

Se acontecer: **não acumular regras manuais.** Aplicar o procedimento de reset deste INC. Tentativas paliativas só pioram o estado e dificultam o diagnóstico.

---

## Lições

**Não confiar em INCs antigos sem revalidar o sintoma exato.** O INC anterior (`INC-PC-2026-05-21-libvirt-virbr0-docker-forward-drop.md`, agora deletado) descrevia um sintoma parecido com causa raiz errada. A solução proposta lá (regras manuais no `DOCKER-USER`) funcionou em alguma instalação anterior por coincidência ou por estado diferente do sistema, e não funcionou nesta sessão. Quando uma "solução conhecida" não resolve, é sinal pra parar de aplicar mais regras e investigar a fundo.

**`iptables -L` mostra apenas uma fração das regras quando o backend é nftables.** Tabelas como `libvirt_network`, `docker-bridges` e `inet firewalld` só aparecem em `nft list ruleset` / `nft list tables`. Mesmo regras MASQUERADE que existem em `nft` podem não aparecer em `iptables -L -t nat -L POSTROUTING`. Em sistemas Fedora modernos, **`nft list ruleset` é a fonte de verdade**, não `iptables-save`.

**Flush de chains não remove tabelas.** `nft flush table` esvazia, mas a tabela permanece em memória. Mesmo `dnf remove docker-ce` não remove as chains que o Docker criou — elas ficam órfãs até `nft delete table` explícito ou reboot. Isso explica por que "limpezas parciais" do firewall ficavam com regras fantasmas e tornavam o diagnóstico mais confuso.

**Quando há suspeita de múltiplos componentes interferindo, o caminho mais rápido é reset estrutural, não tentativa-e-erro.** Foram ~8 horas de tentativas paliativas vs ~1 hora de reset completo. O reset assustava por causa do Tailscale (acesso à VPS), mas era recuperável com `systemctl restart tailscaled` em segundos. Tentar preservar estado a todo custo gastou muito mais tempo do que o reset gastaria.

**`tcpdump` simultâneo em múltiplas interfaces é o diagnóstico mais valioso.** Comparar o que aparece em `vnet0`, `virbr0` e `enp4s0` durante uma tentativa de conexão da VM revelou em segundos onde o pacote sumia. Isso deveria ser o **primeiro** passo de diagnóstico de NAT/forward, não o último.

---

## Hipóteses descartadas (pra referência futura)

Durante a investigação, várias hipóteses foram levantadas e descartadas. Documento aqui pra que ninguém (eu mesmo no futuro) perca tempo testando de novo:

| Hipótese | Por que foi descartada |
|---|---|
| Docker FORWARD policy DROP é a causa raiz | Era um sintoma. Removendo Docker, problema persistiu |
| Tailscale interferindo via `ts-forward` | `tailscale down` não alterou o sintoma |
| Falta de masquerade na zona libvirt | Adicionado masquerade, sintoma não mudou |
| Falta de forward na zona libvirt | Adicionado forward, sintoma mudou de drop pra reject mas continuou bloqueado |
| Policies cross-zone faltando no firewalld | Criadas duas policies, sintoma não resolveu |
| ICMP tratado separadamente | TCP/UDP também falhavam, não era específico de ICMP |
| Cache DNS interno do AdGuard via Tailscale | DNS direto pra 1.1.1.1 também falhava |
| Bridge `virbr0` em estado quebrado | `ip link show virbr0` mostrava UP correto |

---

## Arquivos modificados/criados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/firewalld/firewalld.conf` | Mantido | `FirewallBackend=nftables` (já era default) |
| `/etc/libvirt/network.conf` | Modificado | `firewall_backend = "nftables"` descomentado |
| `/etc/docker/daemon.json` | Criado | `"firewall-backend": "nftables"` + DNS + log opts |

---

## Referências

- Setup completo (procedimento limpo, sem o trauma): `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md`
- ADR de decisão de backend: `ADR-PC-2026-05-22-nftables-backend-unificado-firewall.md`
- ADR de isolamento de redes: `ADR-PC-2026-05-22-libvirt-rede-isolada-do-docker.md`
- Cheat sheet de diagnóstico: `REF-PC-2026-05-22-diagnostico-rede-nftables-fedora.md`
- Documentação Docker sobre nftables backend: https://docs.docker.com/engine/network/firewall-nftables/
- Documentação libvirt sobre firewall_backend: https://libvirt.org/firewall.html
