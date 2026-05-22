# SETUP: QEMU/KVM + libvirt convivendo com Docker no Fedora 44

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, KDE Plasma 6, Wayland

---

## Contexto

Setup de virtualização baseada em KVM no PC pessoal, num host que também roda Docker e Tailscale. A coexistência desses três componentes (libvirt, Docker, Tailscale) no Fedora moderno exige atenção a um ponto não-óbvio: **forçar todos a usar `nftables` nativo em vez do modo de compatibilidade `iptables-nft`**, pra evitar conflito de regras na mesma tabela compartilhada (`ip filter`).

Este documento descreve o procedimento limpo. A história de como esse problema foi descoberto (com várias horas de diagnóstico em tentativas que não funcionaram) está em `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md`.

Decisões arquiteturais relacionadas:
- `ADR-PC-2026-05-22-nftables-backend-unificado-firewall.md` — backend nftables unificado
- `ADR-PC-2026-05-22-libvirt-rede-isolada-do-docker.md` — VMs e containers em redes isoladas

Incidentes relacionados:
- `INC-PC-2026-05-22-kvm-amd-svm-desativado-bios.md` — SVM precisa estar habilitado na BIOS antes de tudo
- `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md` — conflito de backends

---

## Pré-requisitos

- CPU com suporte a virtualização habilitado na BIOS (Intel VT-x ou AMD-V/SVM)
- Em placas Gigabyte AMD: `Tweaker → Advanced CPU Settings → SVM Mode → Enabled`
- IP forwarding habilitado no kernel (default no Fedora)
- firewalld instalado com `FirewallBackend=nftables` (default no Fedora 44)

---

## Fase 0 — Pré-flight

### Confirmar virtualização do CPU

```bash
# Flag svm (AMD) ou vmx (Intel) deve aparecer
grep -E 'vmx|svm' /proc/cpuinfo | head -1

# Versão do Fedora e kernel
cat /etc/fedora-release
uname -r
```

A presença da flag **não garante que esteja habilitado** — apenas que o CPU suporta. Confirmação real vem ao tentar carregar o módulo `kvm_amd`/`kvm_intel`.

### Confirmar backend do firewalld

```bash
sudo grep -i "^FirewallBackend" /etc/firewalld/firewalld.conf
```

Esperado: `FirewallBackend=nftables`. Se for diferente, corrigir e reiniciar firewalld antes de prosseguir.

### Confirmar IP forwarding

```bash
sysctl net.ipv4.ip_forward
```

Esperado: `net.ipv4.ip_forward = 1`. Se for `0`:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-libvirt-forward.conf
sudo sysctl -p /etc/sysctl.d/99-libvirt-forward.conf
```

### Mapear componentes já instalados

```bash
# Docker está instalado?
systemctl is-active docker 2>/dev/null || echo "docker não ativo"
docker --version 2>/dev/null || echo "docker não instalado"

# Subnets Docker em uso (verificar colisão com 192.168.122.0/24 do libvirt)
docker network ls 2>/dev/null
docker network inspect $(docker network ls -q) 2>/dev/null | grep -E '"Subnet"|"Name"'

# Tailscale ativo?
sudo tailscale status | head -5

# libvirt já instalado?
rpm -q libvirt-daemon qemu-kvm virt-manager 2>/dev/null

# Tabelas nftables atuais
sudo nft list tables
```

**Anotar:**
- Interface externa default (`ip route | grep ^default`)
- Subnets Docker em uso (se houver)
- Quais componentes já estão presentes

---

## Fase 1 — Instalar virtualização

```bash
sudo dnf install -y @virtualization
```

Pacotes instalados: `qemu-kvm`, `libvirt-daemon`, `libvirt-client`, `virt-install`, `virt-manager`, `virt-viewer`.

---

## Fase 2 — Configurar backend nftables nativo do libvirt

**Este passo é crítico e deve ser feito ANTES de iniciar o libvirtd.** Se o libvirt subir com o backend padrão (`iptables-nft`), ele vai criar regras na tabela `ip filter` compartilhada, gerando potencial conflito com Tailscale ou outros componentes.

```bash
sudo nano /etc/libvirt/network.conf
```

Descomentar (ou adicionar) a linha:

```
firewall_backend = "nftables"
```

Confirmar:

```bash
sudo grep -E "^firewall_backend" /etc/libvirt/network.conf
```

Esperado: `firewall_backend = "nftables"` (sem `#`).

---

## Fase 3 — Habilitar libvirt e validar KVM

```bash
sudo systemctl enable --now libvirtd

# Adicionar usuário aos grupos (efeito após logout/login)
sudo usermod -aG libvirt,kvm $USER

# Validação completa
sudo virt-host-validate qemu
```

**Esperado em `virt-host-validate`:**
- "verificação por virtualização de hardware" → `PASSAR (SVM)` ou `PASSAR (VMX)`
- "/dev/kvm existe" → `PASSAR`
- "/dev/kvm está acessível" → `PASSAR`
- Warnings em SEV/SEV-ES/TDX são esperados e irrelevantes

**Se "virtualização de hardware" der FAIL** com `SVM disabled (by BIOS) in MSR_VM_CR` no `dmesg`, é SVM desativado na BIOS. Ver `INC-PC-2026-05-22-kvm-amd-svm-desativado-bios.md`.

---

## Fase 4 — Validar tabela libvirt_network criada pelo libvirt

Com `firewall_backend = "nftables"`, o libvirt cria sua **própria tabela** `libvirt_network` no nftables, em vez de injetar regras em `ip filter`.

```bash
# Confirmar que a tabela existe
sudo nft list tables | grep libvirt_network

# Inspecionar conteúdo (deve ter chain forward + guest_nat com MASQUERADE)
sudo nft list table ip libvirt_network | head -30
```

**Esperado:** uma chain `guest_nat` com regras MASQUERADE pra `192.168.122.0/24`, mais chains `guest_input`, `guest_output`, `guest_cross`, `forward`.

### Rede default

```bash
# A rede default já deve estar ativa após libvirtd subir
sudo virsh net-list --all

# Verificar virbr0
ip addr show virbr0
```

Esperado: rede `default` ativa, `virbr0` com IP `192.168.122.1/24`.

---

## Fase 5 — Teste com VM mínima (Alpine)

Sem VM testando, nada está provado. Alpine Linux é leve (~50MB) e perfeita pra validação.

### Baixar ISO

```bash
cd /tmp
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-virt-3.20.3-x86_64.iso
sudo mv alpine-virt-3.20.3-x86_64.iso /var/lib/libvirt/images/
sudo chown qemu:qemu /var/lib/libvirt/images/alpine-virt-3.20.3-x86_64.iso
```

### Criar VM via virt-install

```bash
sudo virt-install \
  --name alpine-test \
  --memory 512 \
  --vcpus 1 \
  --disk size=2,format=qcow2 \
  --cdrom /var/lib/libvirt/images/alpine-virt-3.20.3-x86_64.iso \
  --os-variant alpinelinux3.18 \
  --network network=default \
  --graphics spice \
  --noautoconsole
```

### Abrir console e validar

```bash
sudo virt-viewer alpine-test
```

Login `root` (sem senha). No prompt do Alpine:

```sh
# Configurar interfaces (Alpine live não configura automaticamente)
setup-interfaces -ar

# Validar
ip addr                   # deve ter IP 192.168.122.x
ping -c 3 192.168.122.1   # gateway
ping -c 3 8.8.8.8         # internet por IP
nc -zv 1.1.1.1 443        # TCP direto
ping -c 3 google.com      # DNS + ICMP
```

Tudo deve passar com 0% de packet loss.

---

## Fase 6 — Instalar Docker com nftables nativo

**Só executar esta fase após validar que a VM tem internet (Fase 5).** Isso permite isolar problemas: se algo quebrar após instalar Docker, sabemos que foi o Docker.

### Adicionar repositório oficial

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
```

### Instalar (sem iniciar ainda)

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Configurar daemon.json ANTES do primeiro start

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "firewall-backend": "nftables",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["1.1.1.1", "8.8.8.8"]
}
EOF
```

**Explicação das opções:**

- `firewall-backend: nftables` — Docker 29+ tem suporte experimental a nftables nativo. Cria tabela `docker-bridges` própria, sem injetar em `ip filter`. Sem isso, o conflito do INC volta.
- `log-driver` + `log-opts` — limita logs a 10MB × 3 arquivos por container. Previne enchimento do disco em servidores que rodam por muito tempo.
- `dns` — DNS upstream pros containers. Previne `cURL error 6` causado pelo resolver interno `127.0.0.11` sem upstream (já visto em `FIX-SRV-2026-04-24-nextcloud-aio-docker-dns.md`).

### Iniciar Docker

```bash
sudo systemctl enable --now docker
sleep 3
sudo systemctl status docker | head -10

sudo docker --version
sudo docker run --rm hello-world
```

---

## Fase 7 — Validar coexistência

### Tabelas nftables — cada componente em sua casa

```bash
sudo nft list tables
```

**Esperado (com Tailscale + libvirt + Docker + firewalld):**

```
table ip filter           ← Tailscale
table ip6 filter          ← Tailscale IPv6
table ip nat              ← Tailscale postrouting
table ip6 nat
table ip mangle           ← sistema
table ip6 mangle
table inet firewalld      ← firewalld (zonas, policies)
table ip libvirt_network  ← libvirt (NAT 192.168.122.0/24)
table ip6 libvirt_network
table ip docker-bridges   ← Docker (NAT 172.17.0.0/16)
table ip6 docker-bridges
```

### Confirmar isolamento

```bash
# Docker NÃO deve ter regras na tabela ip filter (compartilhada com Tailscale)
sudo nft list table ip filter | grep -i docker || echo "OK: Docker isolado"

# libvirt também não
sudo nft list table ip filter | grep -i libvirt || echo "OK: libvirt isolado"
```

Esperado: ambos retornam "OK: ... isolado".

### Validar que VM continua com internet

Com Docker rodando, **na VM Alpine**:

```sh
ping -c 3 8.8.8.8
nc -zv 1.1.1.1 443
ping -c 3 google.com
```

Tudo deve continuar funcionando.

---

## Estado final

```
Backend de firewall do sistema   : nftables nativo
firewalld                         : FirewallBackend=nftables, ativo
libvirt                           : firewall_backend = "nftables", tabela libvirt_network própria
Docker                            : "firewall-backend": "nftables", tabela docker-bridges própria
Tailscale                         : iptables-nft (mas em tabelas dedicadas, sem conflito)
IP forwarding                     : ativo persistentemente
Rede libvirt default              : virbr0 (192.168.122.0/24), NAT, autostart=yes
Bridge Docker default             : docker0 (172.17.0.0/16), NAT
Coexistência VMs <-> containers   : redes isoladas (ver ADR)
Grupo do usuário                  : libvirt, kvm
```

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/libvirt/network.conf` | Modificado | `firewall_backend = "nftables"` descomentado |
| `/etc/docker/daemon.json` | Criado | Backend nftables + DNS upstream + log limits |
| `/etc/sysctl.d/99-libvirt-forward.conf` | Criado (se necessário) | Persistir IP forwarding |
| Grupo `libvirt` no `/etc/group` | Modificado | Usuário adicionado |
| Grupo `kvm` no `/etc/group` | Modificado | Usuário adicionado |
| `/var/lib/libvirt/images/` | Populado | ISOs e discos das VMs |

---

## Referência rápida

```bash
# Validar host pra KVM
sudo virt-host-validate qemu

# Listar tabelas (cada componente deve ter a sua)
sudo nft list tables

# Listar VMs
sudo virsh list --all

# Estado da rede default
sudo virsh net-list --all

# Counters de MASQUERADE do libvirt (devem subir conforme VMs usam internet)
sudo nft list table ip libvirt_network | grep masquerade

# Tabela Docker
sudo nft list table ip docker-bridges | head -20

# Logs
sudo journalctl -u libvirtd -f
sudo journalctl -u docker -f
```

---

## Troubleshooting

Se uma VM nova ficar sem internet:

1. Primeiro verificar isolamento das tabelas (`sudo nft list tables` — deve ter `libvirt_network` e `docker-bridges` separadas)
2. Conferir `firewall_backend = "nftables"` em `network.conf` e `firewall-backend: nftables` em `daemon.json`
3. Counters de MASQUERADE subindo em `libvirt_network` quando VM tenta sair?
4. `tcpdump -i virbr0` e `tcpdump -i <iface externa>` simultâneos — onde o pacote some?
5. **Não acumular regras manuais.** Se algo está errado, é problema de backend, não de regra faltando.

Ver `REF-PC-2026-05-22-diagnostico-rede-nftables-fedora.md` pra cheat sheet completo de diagnóstico.

Ver `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md` pra história completa de por que essa abordagem é necessária.
