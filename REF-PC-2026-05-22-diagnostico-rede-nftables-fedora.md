# REF: Diagnóstico de rede em ambiente nftables (Fedora 44+)

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, nftables nativo, libvirt + Docker + Tailscale + firewalld

---

## Princípios

No Fedora moderno, **`nft list ruleset` é a fonte da verdade**, não `iptables-save`. `iptables -L` mostra apenas o que está em iptables-nft (camada de compatibilidade) e ignora tabelas nftables nativas como `libvirt_network`, `docker-bridges`, e `inet firewalld`.

Diagnóstico de problemas de rede deve sempre começar pelo nftables.

---

## Mapeamento mental — quem escreve onde

Em um setup saudável (após `ADR-PC-2026-05-22-nftables-backend-unificado-firewall.md`):

| Tabela nftables | Quem escreve | Propósito |
|---|---|---|
| `ip filter` | Tailscale | chains `ts-input`, `ts-forward` |
| `ip6 filter` | Tailscale | chains `ts-input`, `ts-forward` (IPv6) |
| `ip nat` | Tailscale | chain `ts-postrouting` |
| `ip6 nat` | Tailscale | (IPv6) |
| `ip mangle` | sistema | mangling default |
| `ip6 mangle` | sistema | (IPv6) |
| `inet firewalld` | firewalld | chains de zonas e policies |
| `ip libvirt_network` | libvirt | NAT pra 192.168.122.0/24 |
| `ip6 libvirt_network` | libvirt | (IPv6) |
| `ip docker-bridges` | Docker | NAT pra bridges Docker |
| `ip6 docker-bridges` | Docker | (IPv6) |

Se aparecer uma tabela diferente, é resíduo (de instalação anterior, package que rodou e foi removido, etc.). Limpeza: `sudo nft delete table <família> <nome>`.

---

## Comandos essenciais

### Visão geral

```bash
# Lista todas as tabelas nftables ativas
sudo nft list tables

# Ruleset completo
sudo nft list ruleset

# Listar todas as chains (uma por linha, ordenado)
sudo nft list ruleset | grep -E "^[[:space:]]*chain " | sort -u
```

### Inspecionar tabela específica

```bash
sudo nft list table ip libvirt_network
sudo nft list table ip docker-bridges
sudo nft list table inet firewalld
sudo nft list table ip filter        # Tailscale
```

### Counters subindo? (ver tráfego em tempo real)

```bash
# Ver counters do MASQUERADE do libvirt subirem enquanto VM tenta acessar internet
watch -n 1 "sudo nft list table ip libvirt_network | grep masquerade"

# Counters do Docker
watch -n 1 "sudo nft list table ip docker-bridges | grep -E 'counter|jump'"

# Counter da policy DROP do FORWARD (deve estar zerado em setup saudável)
watch -n 1 "sudo nft list chain ip filter FORWARD"
```

### Buscar por uma rede específica

```bash
# Onde tem regras pra 192.168.122.0/24?
sudo nft list ruleset | grep -E "192.168.122|virbr"

# Onde tem regras pra 172.17.0.0/16?
sudo nft list ruleset | grep -E "172.17.0|docker0"

# Onde tem masquerade?
sudo nft list ruleset | grep -i masquerade
```

---

## Estado do firewalld

```bash
# Estado geral
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones

# Zonas e policies
sudo firewall-cmd --list-all-zones | head -40
sudo firewall-cmd --list-all-policies

# Zona específica
sudo firewall-cmd --zone=libvirt --list-all
sudo firewall-cmd --zone=FedoraWorkstation --list-all

# Backend (deve ser nftables)
sudo grep -i "^FirewallBackend" /etc/firewalld/firewalld.conf

# Aplicar mudanças após editar
sudo firewall-cmd --reload
```

---

## Estado do libvirt

```bash
# Backend (deve ser nftables)
sudo grep "^firewall_backend" /etc/libvirt/network.conf

# Redes definidas
sudo virsh net-list --all

# XML da rede default
sudo virsh net-dumpxml default

# Bridge virbr0 ativa?
ip addr show virbr0
ip -d link show virbr0

# VMs ativas
sudo virsh list --all

# Validação do host pra KVM
sudo virt-host-validate qemu
```

---

## Estado do Docker

```bash
# Backend configurado
sudo grep -E "firewall-backend|iptables" /etc/docker/daemon.json

# Versão (precisa ser 29+ pra suporte a nftables nativo)
sudo docker --version

# Redes
docker network ls
docker network inspect $(docker network ls -q) | grep -E '"Name"|"Subnet"'

# Containers
docker ps -a
```

---

## Diagnóstico de "VM/container sem internet"

### Passo 1 — Confirmar que as tabelas existem

```bash
sudo nft list tables
```

**Esperado:** ver `libvirt_network`, `docker-bridges`, `inet firewalld`, etc.

**Se faltar `libvirt_network`:** libvirt subiu com backend errado (`iptables-nft` em vez de `nftables`). Conferir `/etc/libvirt/network.conf` e reiniciar `libvirtd`.

**Se faltar `docker-bridges`:** Docker subiu com backend errado. Conferir `/etc/docker/daemon.json` e reiniciar `docker`.

### Passo 2 — Confirmar isolamento

```bash
# Docker NÃO deve ter regras na tabela ip filter (do Tailscale)
sudo nft list table ip filter | grep -i docker

# libvirt também não
sudo nft list table ip filter | grep -i libvirt
```

**Esperado:** ambos retornam vazio (não tem nada Docker/libvirt em `ip filter`).

**Se aparecer algo Docker em `ip filter`:** Docker está em modo `iptables-nft`. Migrar pra `nftables` (ver SETUP).

### Passo 3 — Counter de MASQUERADE subindo?

Em paralelo, na VM/container, gerar tráfego:
```sh
# Na VM/container
ping -c 5 8.8.8.8
```

No host:
```bash
sudo nft list table ip libvirt_network | grep masquerade
# Counters de packets devem subir
```

**Se subir mas VM continua sem internet:** problema no caminho de retorno. Vai pro Passo 4.

**Se não subir:** pacote nem está saindo da VM, problema na bridge ou no DHCP. Conferir `ip addr` dentro da VM.

### Passo 4 — tcpdump simultâneo (o ouro do diagnóstico)

Três terminais separados:

```bash
# Terminal 1 — interface da VM no host (vnet0, vnet1, etc — depende da VM)
# Descobrir nome: ip link show | grep vnet
sudo tcpdump -i vnet0 -nn 'host 1.1.1.1 or icmp'

# Terminal 2 — bridge libvirt
sudo tcpdump -i virbr0 -nn 'host 1.1.1.1 or icmp'

# Terminal 3 — interface externa
sudo tcpdump -i enp4s0 -nn 'host 1.1.1.1'
```

Na VM, dispara:
```sh
timeout 5 bash -c 'cat < /dev/tcp/1.1.1.1/443'
```

**Análise:**
- Pacote aparece em **vnet0** mas não em **virbr0**: problema entre vnet e bridge. Bridge mal configurada ou XML da VM com problema.
- Aparece em **virbr0** mas não em **enp4s0**: forward bloqueado. Investigar regras de FORWARD/policies.
- Aparece nas **três interfaces na ida**, mas o retorno (SYN-ACK) chega só em **enp4s0**: problema no caminho de retorno. Geralmente é a chain firewalld da zona externa rejeitando, ou conntrack não trackeou direito.
- Aparece em **enp4s0** com IP da VM (`192.168.122.x`) em vez do IP do host: MASQUERADE não aplicou. Verificar `nft list table ip libvirt_network | grep masquerade`.

### Passo 5 — Se ainda assim não resolver

Não acumule regras manuais. Se o estado está suspeito (muitas tentativas anteriores), o caminho é **reset estrutural**:

```bash
# Backup
mkdir -p ~/network-backup-$(date +%Y%m%d-%H%M%S) && cd $_
sudo nft list ruleset > nft-before.txt

# Procedimento de reset completo em
# INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md
```

---

## Trabalhando com tabelas e chains nftables

### Deletar tabela (limpeza)

```bash
# Atenção: deletar libvirt_network ou docker-bridges quebra esses componentes
# até reiniciar seus serviços, que recriam as tabelas
sudo nft delete table ip <nome-da-tabela>
sudo nft delete table ip6 <nome-da-tabela>
```

### Flush de chain (apaga regras mas mantém a chain)

```bash
sudo nft flush chain ip <tabela> <chain>
```

### Trace de pacote (debug avançado)

```bash
# Habilita trace em uma chain específica
sudo nft add rule ip filter FORWARD meta nftrace set 1

# Em paralelo, em outro terminal
sudo nft monitor trace

# Disparar o pacote a investigar
# Output mostra a passagem do pacote por todas as regras

# Desabilitar (cuidado, vai gerar log no kernel)
sudo nft flush chain ip filter FORWARD
# (e reaplicar regras desejadas, se for chain customizada)
```

---

## Compatibilidade com iptables (quando ainda for relevante)

```bash
# Ver com lente "iptables" o que está em iptables-nft
sudo iptables -L -n -v --line-numbers
sudo iptables -t nat -L -n -v --line-numbers

# Backend do alternative
sudo alternatives --display iptables
# Esperado: link aponta pra /usr/bin/iptables-nft

# Forçar alternative pra nft (se algum bug fizer voltar pra legacy)
sudo alternatives --set iptables /usr/sbin/iptables-nft
sudo alternatives --set ip6tables /usr/sbin/ip6tables-nft
```

**Cuidado:** muito do que aparece em `iptables -L` no Fedora é só uma visão parcial. Pra ver tudo, use `nft`.

---

## Comandos de teste rápido (de dentro de uma VM ou container)

```bash
# Conectividade básica
ping -c 3 192.168.122.1     # gateway libvirt (da VM)
ping -c 3 172.17.0.1        # gateway Docker (de um container)
ping -c 3 8.8.8.8           # internet por IP
ping -c 3 google.com        # DNS + internet

# TCP direto (sem ICMP, útil quando ICMP é filtrado)
nc -zv 1.1.1.1 443
timeout 5 bash -c 'cat < /dev/tcp/1.1.1.1/443' && echo OK || echo FAIL

# DNS específico
nslookup google.com 1.1.1.1
dig @1.1.1.1 google.com

# Sem dig/curl/nc (Ubuntu mínimo):
wget -v --timeout=5 --tries=1 https://1.1.1.1 -O /dev/null
getent hosts google.com
```

---

## Quando algo está esquisito — checklist rápido

1. `sudo nft list tables` — todas as tabelas esperadas estão lá?
2. `sudo systemctl is-active firewalld libvirtd docker tailscaled` — todos ativos?
3. `sudo virsh net-list --all` — rede default ativa?
4. `ip addr show virbr0` — bridge tem IP `192.168.122.1`?
5. `sysctl net.ipv4.ip_forward` — está em `1`?
6. tcpdump simultâneo em vnet/virbr0/enp4s0 — onde o pacote some?

Se nada disso revelar o problema, é hora de revisar `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md` e considerar reset estrutural.

---

## Referências

- `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md` — setup completo
- `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md` — história da causa raiz
- `ADR-PC-2026-05-22-nftables-backend-unificado-firewall.md` — decisão de backend
- Documentação nftables: https://wiki.nftables.org/
- Documentação Docker nftables: https://docs.docker.com/engine/network/firewall-nftables/
