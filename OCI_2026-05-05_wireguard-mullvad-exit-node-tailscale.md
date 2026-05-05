# Setup: WireGuard (Mullvad VPN) como saída do exit node Tailscale

- **Data:** 2026-05-05
- **Local:** VM Oracle Cloud ({{OCI_HOSTNAME}})
- **Sistema:** Ubuntu 24.04 ARM, Tailscale, WireGuard

---

## Contexto / Problema

O Android permite apenas uma VPN ativa por vez. O Tailscale é necessário para
acessar os serviços que rodam na VM (Nextcloud, AdGuard, etc.), mas sem uma VPN
de saída, o tráfego de internet do celular sai pelo IP da operadora.

A solução foi configurar a VM como exit node do Tailscale e fazer o tráfego de
internet sair pelo Mullvad VPN via WireGuard. O fluxo:

```
celular → [Tailscale] → VM → [WireGuard/Mullvad VPN] → internet
```

O desafio é que a VM é servidor e cliente ao mesmo tempo — ela precisa:
- Responder requisições aos próprios serviços (Nextcloud, etc.) pelo Tailscale
- Encaminhar tráfego de internet dos dispositivos para o Mullvad VPN
- Manter seu próprio tráfego de gerenciamento (SSH, atualizações) fora do túnel VPN

---

## Por que Mullvad e não Proton VPN

O Proton VPN foi a primeira escolha mas apresentou dois problemas:

**1. CLI não funciona em ambiente headless.** A CLI do Proton depende de
`gnome-keyring` e `NetworkManager` — componentes de desktop. A autenticação
funciona mas `protonvpn connect` falha silenciosamente em servidores sem GUI.

**2. Servidores "BR" fisicamente em Miami.** Os servidores listados como
"BR-SP" pelo Proton estavam fisicamente em Miami (AS9009 M247), resultando em
~200ms de latência e 30% de packet loss. Verificado via `curl https://ipinfo.io/`.

O Mullvad tem política explícita de não usar servidores virtuais — todos os
servidores estão fisicamente onde estão listados. Os servidores BR do Mullvad
em São Paulo apresentaram 0.5ms de latência da Oracle SP.

| Provedor | Servidor | Localização real | Latência | Packet loss |
|---|---|---|---|---|
| Proton "BR-SP" | Miami 🇺🇸 | 197ms | 30% |
| Mullvad BR (Datapacket) | São Paulo 🇧🇷 | 0.5ms | 0% |

---

## Causa Raiz do desafio de configuração

O `wg-quick` por padrão vira o gateway padrão da máquina inteira. Numa VM
servidor isso quebra o acesso SSH porque o tráfego de gerenciamento passa pelo
túnel e o provedor VPN não sabe rotear de volta para a rede Tailscale.

A distinção correta: tráfego destinado à própria VM vai para `INPUT` do kernel,
nunca passa pelo `FORWARD`. Só o tráfego que o Tailscale está encaminhando para
internet passa pelo `FORWARD`. O critério de roteamento deve ser a interface de
entrada, não a faixa de IP de origem.

Tentativas que falharam durante o desenvolvimento:
- `AllowedIPs` com ranges excluindo Tailscale — o endpoint VPN fica sem rota
- `ip rule add from 100.64.0.0/10` — a VM tem IP Tailscale nesse range, as
  respostas SSH são roteadas pelo túnel e o acesso quebra
- `PostUp` com `ip rule add fwmark` — correto mas incompleto sem MASQUERADE

---

## Solução

Três elementos combinados:

**1. `Table = off`** — WireGuard sobe a interface sem mexer nas rotas do sistema.

**2. Policy routing via `iif tailscale0`** — tabela de rotas separada (51820)
onde o gateway é o túnel Mullvad. A regra `iif tailscale0` afeta apenas pacotes
que chegam pela interface Tailscale sendo forwardados — não afeta respostas que
a VM gera localmente.

**3. MASQUERADE + MSS clamping** — MASQUERADE obrigatório para que o Mullvad
aceite os pacotes (IPs `100.x.x.x` são CGNAT). MSS clamping previne timeout em
HTTPS causado pelo duplo encapsulamento Tailscale+WireGuard.

---

## Como configurar

### Pré-requisitos

- IP forwarding ativo (feito pelo `provision.sh`)
- Tailscale instalado e configurado como exit node
- Arquivo `.conf` WireGuard gerado em `mullvad.net/en/account/wireguard-config`
  - Opções ao gerar: Multihop desligado, Kill switch desligado, Content blocking desligado
  - Escolher servidor BR — provider Datapacket (São Paulo, 0.5ms da Oracle SP)
  - Nome da interface deve ter no máximo 15 caracteres

### Instalar WireGuard

```bash
sudo apt install wireguard resolvconf -y
```

### Criar o arquivo de configuração

```bash
sudo nano /etc/wireguard/wg-mull-br.conf
sudo chmod 600 /etc/wireguard/wg-mull-br.conf
```

Conteúdo (substituir placeholders pelos valores do `.conf` gerado no painel Mullvad):

```ini
[Interface]
PrivateKey = {{WG_MULLVAD_PRIVATE_KEY}}
Address    = {{WG_MULLVAD_ADDRESS}}
Table      = off

# NAT obrigatório — mascara IPs Tailscale (100.64.0.0/10) para o Mullvad aceitar
PostUp   = iptables -t nat -A POSTROUTING -o %i -s 100.64.0.0/10 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o %i -s 100.64.0.0/10 -j MASQUERADE

# MSS clamping — evita timeout em HTTPS com duplo encapsulamento Tailscale+WireGuard
PostUp   = iptables -t mangle -A FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
PostDown = iptables -t mangle -D FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# Roteamento: tráfego que ENTRA pela interface Tailscale usa tabela 51820
PostUp   = ip rule add iif tailscale0 table 51820
PostDown = ip rule del iif tailscale0 table 51820

# Tabela 51820: internet sai pelo Mullvad, range Tailscale interno fica local
PostUp   = ip route add default dev %i table 51820
PostUp   = ip route add 100.64.0.0/10 dev tailscale0 table 51820
PostDown = ip route del default dev %i table 51820
PostDown = ip route del 100.64.0.0/10 dev tailscale0 table 51820

[Peer]
PublicKey = {{WG_MULLVAD_PEER_PUBLIC_KEY}}
AllowedIPs = 0.0.0.0/0
Endpoint   = {{WG_MULLVAD_ENDPOINT_IP}}:51820
PersistentKeepalive = 25
```

> Remover a linha `DNS` do `.conf` gerado pelo Mullvad — a VM usa seu próprio
> DNS. Quando o AdGuard subir (Fase 2), ele assume esse papel.

### Subir e habilitar no boot

```bash
sudo wg-quick up wg-mull-br
sudo systemctl enable wg-quick@wg-mull-br
```

### Trocar de servidor

```bash
sudo wg-quick down wg-mull-br
sudo ip route del 100.64.0.0/10 table 51820 2>/dev/null
sudo ip rule del iif tailscale0 table 51820 2>/dev/null
sudo wg-quick up wg-mull-us   # ou outro servidor
```

> As rotas órfãs precisam ser limpas manualmente antes de subir outro servidor —
> o PostDown não as remove se o tunnel cair de forma inesperada.

---

## Verificação / Referência rápida

```bash
# Tunnel ativo e com handshake
sudo wg show

# Regras e rotas corretas
ip rule | grep tailscale        # deve mostrar: iif tailscale0 lookup 51820
ip route show table 51820       # deve mostrar: default dev wg-mull-br + 100.64.0.0/10 dev tailscale0
ip route | grep default         # default original via 10.0.0.1 dev enp0s6, intacto

# Tráfego da própria VM — deve continuar sendo IP da Oracle
curl https://api.ipify.org      # retorna {{OCI_PUBLIC_IP}}

# No celular com exit node ativo no Tailscale:
# ifconfig.me — deve mostrar IP Mullvad (169.150.198.x)
# fast.com — deve marcar ~300 Mbps
```

---

## Estado final

```
Interface VM    : enp0s6 (gateway 10.0.0.1, IP Oracle intacto)
Tailscale       : exit node ativo, IP {{OCI_TS_IP}}
WireGuard       : wg-mull-br ativo (Table=off, policy routing via tabela 51820)
Servidor VPN    : Mullvad BR Datapacket, São Paulo ({{WG_MULLVAD_ENDPOINT_IP}})
Tráfego da VM   : sai pela Oracle (SSH, atualizações, serviços)
Tráfego celular : sai pelo Mullvad VPN via WireGuard (~300 Mbps, 0.5ms)
Boot            : wg-quick@wg-mull-br habilitado via systemctl
```

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/wireguard/wg-mull-br.conf` | Criado | Config WireGuard Mullvad BR (Datapacket SP) |
| `/etc/wireguard/wg-mull-us.conf` | Criado | Config WireGuard Mullvad US (backup) |
| `/etc/wireguard/wg-mull-uk.conf` | Criado | Config WireGuard Mullvad UK (backup) |
