# Setup: WireGuard (Proton VPN) como saída do exit node Tailscale

- **Data:** 2026-05-04
- **Local:** VM Oracle Cloud ({{OCI_HOSTNAME}})
- **Sistema:** Ubuntu 24.04 ARM, Tailscale, WireGuard

---

## Contexto / Problema

O Android permite apenas uma VPN ativa por vez. O Tailscale é necessário para acessar os serviços que rodam na VM (Nextcloud, AdGuard, etc.), mas sem uma VPN de saída, o tráfego de internet do celular sai pelo IP da operadora.

A solução foi configurar a VM como exit node do Tailscale e fazer o tráfego de internet sair pelo Proton VPN via WireGuard. O fluxo desejado:

```
celular → [Tailscale] → VM → [WireGuard/Proton VPN] → internet
```

O desafio é que a VM é servidor e cliente ao mesmo tempo — ela precisa:
- Responder requisições aos próprios serviços (Nextcloud, etc.) pelo Tailscale
- Encaminhar tráfego de internet dos dispositivos para o Proton VPN
- Manter seu próprio tráfego de gerenciamento (SSH, atualizações) fora do túnel VPN

A abordagem padrão do WireGuard (`wg-quick` com `AllowedIPs = 0.0.0.0/0`) captura todo o tráfego da VM e quebra o acesso SSH e o próprio Tailscale.

---

## Causa Raiz

O `wg-quick` por padrão vira o gateway padrão da máquina inteira. Numa VM servidor isso quebra tudo porque o tráfego de gerenciamento (SSH via Tailscale) passa pelo túnel e o Proton não sabe rotear de volta para a rede Tailscale.

A distinção correta não é "de onde veio o tráfego" (origem IP) mas sim "por qual interface chegou sendo forwardado" — tráfego destinado à própria VM vai para `INPUT` do kernel, nunca passa pelo `FORWARD`. Só o tráfego que o Tailscale está encaminhando para internet passa pelo `FORWARD`.

Tentativas que falharam:
- `AllowedIPs` com ranges excluindo Tailscale — o endpoint do Proton também fica sem rota e o túnel não sobe
- `PostUp = ip rule add from 100.64.0.0/10` — a própria VM tem IP Tailscale nesse range, as respostas SSH são roteadas pelo Proton e o acesso quebra
- `PostUp = ip rule add fwmark` via iptables FORWARD — correto conceitualmente mas incompleto sem o MASQUERADE

---

## Solução

Três elementos combinados:

**1. `Table = off`** — WireGuard sobe a interface sem mexer nas rotas do sistema. A VM continua usando `enp0s6` normalmente.

**2. Policy routing via `iif tailscale0`** — uma tabela de rotas separada (51820) onde o gateway é o túnel Proton. A regra `iif tailscale0` manda para essa tabela apenas pacotes que chegam pela interface Tailscale sendo forwardados — não afeta respostas que a VM gera localmente.

**3. MASQUERADE + MSS clamping** — o MASQUERADE é obrigatório para que o Proton aceite os pacotes (IPs `100.x.x.x` são CGNAT, o Proton descartaria sem NAT). O MSS clamping previne timeout em HTTPS causado pelo duplo encapsulamento (Tailscale dentro de WireGuard reduz o MTU efetivo).

---

## Como configurar

### Pré-requisitos

- IP forwarding ativo (feito pelo `provision.sh`)
- Tailscale instalado e configurado como exit node
- Arquivo `.conf` WireGuard gerado no painel do Proton VPN (`account.proton.me` → VPN → WireGuard)
  - Opções ao gerar: Netshield desligado (AdGuard faz esse papel), VPN Accelerator ligado, Moderate NAT desligado

### Instalar WireGuard

```bash
sudo apt install wireguard resolvconf -y
```

### Criar o arquivo de configuração

```bash
sudo nano /etc/wireguard/wg-br-12.conf
sudo chmod 600 /etc/wireguard/wg-br-12.conf
```

Conteúdo do arquivo (substituir placeholders pelos valores reais do `.conf` gerado no painel Proton):

```ini
[Interface]
PrivateKey = {{WG_PRIVATE_KEY}}
Address = {{WG_ADDRESS}}
DNS = {{WG_DNS}}
Table = off

# NAT obrigatório — mascara IPs Tailscale (100.64.0.0/10) para o Proton aceitar
PostUp   = iptables -t nat -A POSTROUTING -o %i -s 100.64.0.0/10 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o %i -s 100.64.0.0/10 -j MASQUERADE

# MSS clamping — evita timeout em HTTPS com duplo encapsulamento Tailscale+WireGuard
PostUp   = iptables -t mangle -A FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
PostDown = iptables -t mangle -D FORWARD -o %i -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# Roteamento: tráfego que ENTRA pela interface Tailscale usa tabela 51820
# iif (incoming interface) — não afeta respostas que a VM gera localmente
PostUp   = ip rule add iif tailscale0 table 51820
PostDown = ip rule del iif tailscale0 table 51820

# Tabela 51820: internet sai pelo Proton, range Tailscale interno fica local
PostUp   = ip route add default dev %i table 51820
PostUp   = ip route add 100.64.0.0/10 dev tailscale0 table 51820
PostDown = ip route del default dev %i table 51820
PostDown = ip route del 100.64.0.0/10 dev tailscale0 table 51820

[Peer]
PublicKey = {{WG_PEER_PUBLIC_KEY}}
AllowedIPs = 0.0.0.0/0
Endpoint = {{WG_ENDPOINT_IP}}:51820
PersistentKeepalive = 25
```

### Subir e habilitar no boot

```bash
# Subir o tunnel
sudo wg-quick up wg-br-12

# Habilitar para subir automaticamente no boot
sudo systemctl enable wg-quick@wg-br-12
```

### Múltiplos servidores (opcional)

Gerar `.conf` para outros servidores (ex: US, UK) no painel do Proton e aplicar o mesmo template acima. Trocar de servidor:

```bash
sudo wg-quick down wg-br-12
sudo wg-quick up wg-us-07
```

Os arquivos `.conf` expiram anualmente — renovar no painel do Proton (botão "Extend") e atualizar os arquivos na VM.

---

## Verificação / Referência rápida

```bash
# Confirmar que as regras e rotas estão corretas após subir
ip rule | grep tailscale        # deve mostrar: iif tailscale0 lookup 51820
ip route show table 51820       # deve mostrar: default dev wg-br-12 + 100.64.0.0/10 dev tailscale0
ip route | grep default         # default original via gateway da Oracle, intacto

# Tráfego da própria VM — deve continuar sendo IP da Oracle
curl https://api.ipify.org

# Status do tunnel WireGuard
sudo wg show

# No celular com exit node ativo no Tailscale:
# abrir ifconfig.me — deve mostrar IP do Proton VPN, não da Oracle
```

---

## Estado final

```
Interface VM   : enp0s6 (gateway 10.0.0.1, IP Oracle intacto)
Tailscale      : exit node ativo, IP {{OCI_TS_IP}}
WireGuard      : wg-br-12 ativo (Table=off, policy routing via tabela 51820)
Tráfego da VM  : sai pela Oracle (SSH, atualizações, serviços)
Tráfego celular: sai pelo Proton VPN via WireGuard
Boot           : wg-quick@wg-br-12 habilitado via systemctl
```

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/wireguard/wg-br-12.conf` | Criado | Config WireGuard servidor BR com policy routing |
| `/etc/wireguard/wg-us-*.conf` | Criado | Config WireGuard servidor US (backup) |
| `/etc/wireguard/wg-uk-*.conf` | Criado | Config WireGuard servidor UK (backup) |
