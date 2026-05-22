# SETUP: QEMU/KVM + libvirt convivendo com Docker no Fedora

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, KDE Plasma 6, Wayland, Docker já instalado e ativo

---

## Contexto

Setup de virtualização baseada em KVM no PC pessoal, com a particularidade de já existir uma stack Docker rodando (n8n e bridges padrão). A coexistência entre libvirt e Docker no mesmo host exige atenção a duas armadilhas: SVM precisa estar habilitado na BIOS, e o Docker reescreve a chain `FORWARD` do iptables com policy `DROP`, o que quebra o tráfego de saída das VMs se nada for feito.

Este documento é o roteiro completo para refazer o setup do zero. Cada fase tem verificação antes e depois — não aplicar nada sem confirmar o estado atual primeiro.

Incidentes relacionados:
- `INC-PC-2026-05-21-libvirt-virbr0-docker-forward-drop.md` — VMs sem internet por causa do Docker
- `INC-PC-2026-05-22-kvm-amd-svm-desativado-bios.md` — virt-manager reclamando que KVM não está disponível
- `ADR-PC-2026-05-22-libvirt-rede-isolada-do-docker.md` — decisão de manter VMs e containers em redes separadas

---

## Pré-requisitos

- CPU com suporte a virtualização (Intel VT-x ou AMD-V/SVM)
- **SVM habilitado na BIOS** (em placas Gigabyte: Tweaker → Advanced CPU Settings → SVM Mode → Enabled)
- IP forwarding habilitado no kernel (geralmente já vem ativo no Fedora)
- Docker já instalado se for o caso (este setup foi feito com Docker rodando)

---

## Fase 0 — Pré-flight: entender o estado atual

### Confirmar virtualização disponível

```bash
# Flag svm (AMD) ou vmx (Intel) deve aparecer
grep -E 'vmx|svm' /proc/cpuinfo | head -1

# Versão do Fedora e kernel
cat /etc/fedora-release
uname -r
```

A flag estar presente no `/proc/cpuinfo` não garante que a virtualização esteja **habilitada** — só que o CPU suporta. A confirmação real vem ao tentar carregar o módulo `kvm_amd`/`kvm_intel` (Fase 2).

### Mapear estado atual do Docker e da rede

```bash
# Docker ativo?
systemctl is-active docker
docker --version

# Subnets Docker em uso (para detectar colisão com 192.168.122.0/24 da libvirt)
docker network ls
docker network inspect $(docker network ls -q) | grep -E '"Subnet"|"Name"'

# Interface externa default — anotar pra usar nas regras de firewall
ip route | grep '^default'

# libvirt já instalado?
rpm -q libvirt-daemon qemu-kvm virt-manager

# virbr0 já existe?
ip link show virbr0 2>/dev/null || echo "virbr0 não existe ainda"
```

### Mapear firewall atual

```bash
# Estado do firewalld
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all

# Policy da chain FORWARD — Docker força DROP, libvirt sozinho deixa ACCEPT
sudo iptables -L FORWARD -n -v --line-numbers | head -20

# Chain DOCKER-USER (criada pelo Docker, é onde a gente vai mexer)
sudo iptables -L DOCKER-USER -n -v --line-numbers 2>/dev/null
```

**O que olhar:**
- Policy `FORWARD` é `DROP` se Docker está rodando — isso é normal e esperado
- `DOCKER-USER` existe e está vazia (ou tem regras de outros setups)
- Nenhuma subnet Docker pode colidir com `192.168.122.0/24`

---

## Fase 1 — Instalar a stack de virtualização

### Verificar antes

```bash
# Conteúdo do grupo
sudo dnf group info virtualization
```

Confirmar que inclui `qemu-kvm`, `libvirt-daemon`, `virt-install`, `virt-manager`.

### Instalar

```bash
sudo dnf install @virtualization
```

### Verificar depois

```bash
rpm -q libvirt-daemon qemu-kvm virt-manager virt-install
systemctl status libvirtd --no-pager
```

---

## Fase 2 — Ativar libvirt e checar grupo do usuário

### Verificar antes

```bash
groups $USER | tr ' ' '\n' | grep -E 'libvirt|kvm'
```

### Aplicar

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm $USER
```

### Verificar depois

```bash
systemctl is-active libvirtd
sudo virt-host-validate qemu
```

**Esperado em `virt-host-validate`:**
- "verificação por virtualização de hardware" → `PASSAR (SVM)` ou `PASSAR (VMX)`
- "/dev/kvm existe" → `PASSAR`
- "/dev/kvm está acessível" → `PASSAR`
- Warnings em IOMMU sem efeito prático para uso desktop com NAT
- Warning em SEV/SEV-ES/TDX é esperado — recursos de "convidado seguro" não necessários

**Se "virtualização de hardware" der FAIL com `SVM disabled (by BIOS) in MSR_VM_CR`** no `dmesg`, é SVM desativado na BIOS. Ver `INC-PC-2026-05-22-kvm-amd-svm-desativado-bios.md`.

> O `usermod` só tem efeito após **logout completo da sessão**. Não basta abrir um terminal novo.

---

## Fase 3 — Inspecionar a rede default do libvirt

Verificação obrigatória antes de subir a rede — pra detectar colisão de subnet com Docker.

### Verificar antes

```bash
# A rede default existe mas geralmente inicia inativa após instalação
sudo virsh net-list --all

# XML da definição
sudo virsh net-dumpxml default
```

**O que confirmar no XML:**
- `<forward mode='nat'/>` — modo NAT (queremos isso)
- `<bridge name='virbr0' stp='on' delay='0'/>`
- `<ip address='192.168.122.1' netmask='255.255.255.0'>` — esta subnet **não pode** colidir com nenhuma subnet Docker

### Checar colisão com Docker

```bash
docker network inspect $(docker network ls -q) | grep '"Subnet"'
```

Subnets Docker default ficam em `172.17.0.0/16`, `172.18.0.0/16` etc. — sem colisão com `192.168.122.0/24`. Se houver colisão, parar e editar o XML da rede default antes de seguir.

---

## Fase 4 — Subir a rede default

### Aplicar

```bash
sudo virsh net-autostart default
sudo virsh net-start default
```

### Verificar depois

```bash
# Esperado: default | ativo | sim | sim
sudo virsh net-list --all

# Esperado: inet 192.168.122.1/24
ip addr show virbr0

# dnsmasq do libvirt escutando na virbr0
sudo ss -ulnp | grep dnsmasq
```

---

## Fase 5 — Coexistir com Docker (regras de firewall)

Esta é a fase crítica. Sem ela, VMs pingam o gateway `192.168.122.1` mas não saem pra internet, porque o Docker setou `FORWARD policy DROP`.

### Verificar estado atual

```bash
# Policy do FORWARD — esperado: DROP (com Docker ativo)
sudo iptables -L FORWARD -n | head -1

# Chains de FORWARD na ordem que o Docker injeta
sudo iptables -L FORWARD -n -v --line-numbers | head -20

# DOCKER-USER deve existir e provavelmente estar vazia
sudo iptables -L DOCKER-USER -n --line-numbers

# IP forwarding habilitado no kernel
sysctl net.ipv4.ip_forward
```

**Decisão:**
- Se Docker está rodando com `FORWARD policy DROP` → seguir a Fase 5 inteira
- Se Docker não está instalado → pular pra Fase 6 (mas, se instalar Docker depois, retornar pra Fase 5)

### Aplicar regras

```bash
# EXT = interface externa, obtida em "ip route | grep default"
EXT=enp4s0  # ajustar pra sua interface real

# Saída das VMs pra internet
sudo iptables -I DOCKER-USER -i virbr0 -o $EXT -j ACCEPT

# Retorno da internet pras VMs — só pacotes de conexões já iniciadas pelas VMs (conntrack)
sudo iptables -I DOCKER-USER -o $EXT -i virbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

A segunda regra usa conntrack pra ser mais segura que um `ACCEPT` cego — só libera tráfego de retorno de conexões que as VMs iniciaram, nada de tráfego entrante novo da internet pra VM.

### Verificar

```bash
sudo iptables -L DOCKER-USER -n -v --line-numbers
```

Esperado: as 2 regras com ACCEPT.

### Persistir as regras (obrigatório)

Regras `iptables` somem após reboot. Persistência via systemd unit:

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

# Substituir placeholder pela interface real
sudo sed -i "s/EXT_PLACEHOLDER/$EXT/g" /etc/systemd/system/libvirt-docker-bridge.service

sudo systemctl daemon-reload
sudo systemctl enable --now libvirt-docker-bridge.service
sudo systemctl status libvirt-docker-bridge.service --no-pager
```

---

## Fase 6 — Teste real com uma VM mínima

Sem VM, nada está provado. Alpine Linux é leve (~50MB) e perfeita pra validação.

### Baixar ISO

```bash
cd /tmp
curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-virt-3.20.3-x86_64.iso
sudo mv alpine-virt-*.iso /var/lib/libvirt/images/
```

### Criar VM via virt-manager

Abrir virt-manager → Nova VM → apontar pra ISO → 1 vCPU, 512MB RAM, 2GB disco → Rede: NAT, default.

### Verificar dentro da VM

Boot Alpine em modo live (`root`, sem senha):

```bash
# IP atribuído pelo DHCP do libvirt
ip addr

# Gateway aponta pra 192.168.122.1
ip route

# Gateway responde
ping -c 2 192.168.122.1

# Internet responde (este é O teste — falha aqui = Fase 5 falhou)
ping -c 2 8.8.8.8

# DNS funciona
ping -c 2 google.com
```

**Cenários:**
- Tudo passa → setup completo
- Gateway pinga mas `8.8.8.8` não → regra do `DOCKER-USER` não está aplicada; revisar Fase 5
- Nem o gateway pinga → problema na bridge ou na config da VM

---

## Estado final

```
Virtualização    : KVM acelerado por hardware (kvm_amd carregado)
libvirt          : ativo e habilitado no boot
Rede default     : virbr0 (192.168.122.0/24), NAT, autostart=yes
Docker           : continua rodando normalmente, isolado das VMs
Coexistência     : libvirt-docker-bridge.service injeta ACCEPT em DOCKER-USER no boot
Grupo do usuário : libvirt, kvm (após logout/login)
```

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/systemd/system/libvirt-docker-bridge.service` | Criado | Unit que injeta regras ACCEPT em DOCKER-USER pra coexistência libvirt+Docker |
| Grupo `libvirt` no `/etc/group` | Modificado | Usuário adicionado |
| Grupo `kvm` no `/etc/group` | Modificado | Usuário adicionado |
| `/var/lib/libvirt/images/` | Populado | ISOs e discos de VMs |

---

## Referência rápida

```bash
# Validar host pra KVM
sudo virt-host-validate qemu

# Status da rede default
sudo virsh net-list --all
sudo virsh net-dumpxml default

# Verificar regras de coexistência
sudo iptables -L DOCKER-USER -n -v --line-numbers
sudo systemctl status libvirt-docker-bridge.service

# Listar VMs
sudo virsh list --all

# Logs do libvirt
sudo journalctl -u libvirtd -f
```
