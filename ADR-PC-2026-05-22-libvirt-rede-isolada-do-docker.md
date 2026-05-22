# ADR: Rede libvirt isolada do Docker no PC pessoal

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, libvirt + Docker convivendo no mesmo host
- **Status:** Aceita

---

## Contexto

O PC pessoal tem uma stack Docker rodando (n8n e bridges padrão em `172.17.0.0/16` e `172.18.0.0/16`). Ao instalar libvirt para rodar VMs, surgiu a decisão de como integrar (ou não) a rede das VMs com a rede dos containers.

As três opções principais para integração de rede entre libvirt e Docker:

1. **Isolamento total** — VMs e containers em redes separadas, sem rota direta entre eles. Comunicação possível só via interfaces externas (host atuando como roteador, ou via internet).
2. **Bridge física compartilhada (`br0`)** — host expõe uma bridge layer 2 onde tanto VMs quanto containers (em modo macvlan/ipvlan) se conectam, virando hosts de primeira classe na LAN.
3. **Rota explícita entre subnets** — manter redes separadas mas adicionar regras de roteamento e firewall pra permitir comunicação entre `virbr0` (192.168.122.0/24) e bridges Docker.

Considerando o caso de uso atual (VMs descartáveis pra teste, experimentação com OS, sandboxes de desenvolvimento; containers Docker servindo aplicações pessoais como n8n), a complexidade de integração não se justifica.

---

## Decisão

**VMs e containers Docker operam em redes completamente isoladas:**

- VMs usam a rede `default` do libvirt: `virbr0`, subnet `192.168.122.0/24`, modo NAT
- Containers Docker usam suas bridges nativas (`bridge`, `n8n_default`, etc.) em subnets `172.17.x.x` e `172.18.x.x`
- A única regra de firewall que cruza os dois mundos é a injeção em `DOCKER-USER` que libera o tráfego de saída da `virbr0` para a interface externa (`enp4s0`) — necessária pra VMs terem internet (ver `INC-PC-2026-05-21-libvirt-virbr0-docker-forward-drop.md`)
- **Não há rota direta entre VMs e containers.** Se uma VM precisar conversar com um container, o caminho é via o IP do host na rede externa (ou via porta exposta pelo Docker em `localhost`/`0.0.0.0`)

VMs saem para internet via NAT default do libvirt através da `enp4s0`. Containers saem pelo NAT do Docker, também via `enp4s0`. Nenhum dos dois passa pelo outro.

---

## Alternativas consideradas

### Bridge física compartilhada (`br0`)

Criar uma bridge `br0` no host, mover a interface física `enp4s0` para dentro dela, e conectar VMs e containers (via macvlan) nessa bridge. VMs e containers seriam hosts de primeira classe na rede 192.168.1.0/24 (LAN doméstica).

**Vantagens:** comunicação direta L2, IPs reais na LAN, fácil descoberta entre VMs/containers e outros dispositivos da rede.

**Descartada porque:**
- Complexidade de configuração alta (bridge no NetworkManager, ajustes nas duas stacks)
- Risco de quebrar conectividade do próprio host durante o setup
- VMs e containers começam a competir por IPs do DHCP da LAN
- Implicações de segurança: containers passam a estar expostos diretamente na LAN doméstica
- O ganho não compensa o caso de uso atual — VMs são descartáveis e experimentais

### Rota explícita entre `virbr0` e bridges Docker

Manter as redes separadas, mas adicionar regras `iptables` permitindo `FORWARD` direto entre `virbr0` e `docker0` (e bridges customizadas do Docker).

**Vantagens:** isolamento topológico mantido, comunicação possível só onde necessário.

**Descartada porque:**
- Multiplica a superfície de configuração: cada nova rede Docker (cada `docker-compose` que cria uma bridge custom) demanda nova regra
- Aumenta o risco de regressão a cada reinício do Docker (que limpa regras fora do `DOCKER-USER`)
- O caso de uso atual não exige comunicação VM↔container — não há serviço Docker que VMs precisem acessar diretamente

### Rede libvirt com policy routing dedicada (espelhando padrão OCI)

Criar uma segunda rede libvirt isolada (ex.: `vmnet1`, subnet `192.168.100.0/24`) com policy routing dedicada, similar ao setup do OCI com WireGuard/Tailscale documentado em `OCI_2026-05-05_wireguard-mullvad-exit-node-tailscale.md`.

**Considerada mas adiada porque:** não há necessidade atual de VPN nas VMs, e a rede `default` resolve. **Documentado como caminho futuro**: se algum dia surgir necessidade de VMs com saída via VPN (ex.: rodar uma VM para testar geo-bloqueio), criar uma segunda rede libvirt com policy routing dedicado, mantendo a `default` intocada.

---

## Consequências

### Positivas

- Setup simples, fácil de reproduzir e documentar
- Comportamento previsível — a única regra de firewall cross-stack é a do `DOCKER-USER`
- Manutenção mínima: novas redes Docker (cada `docker-compose` que cria uma bridge custom) não exigem regras adicionais
- Segurança por padrão: VMs comprometidas não conseguem chegar nos containers diretamente

### Negativas / Trade-offs

- Se uma VM precisar conversar com um container (ex.: testar uma API exposta pelo n8n), tem que ir pela porta exposta no host (`192.168.122.1:PORTA` a partir da VM, ou via IP da LAN)
- Sem descoberta automática entre VMs e containers — cada lado é uma ilha
- Se eventualmente houver necessidade de comunicação extensa VM↔container, esta decisão precisa ser revista (nova ADR)

### Caminho futuro

Se surgir necessidade de VMs passarem por VPN (Mullvad/Proton), o padrão a seguir é criar uma **segunda rede libvirt** (ex.: `vmnet-vpn`, subnet `192.168.123.0/24`) com policy routing dedicado, espelhando o padrão usado na VM OCI com WireGuard. A rede `default` continua intocada para VMs que saem direto pelo host.

---

## Referências

- `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md` — setup completo que materializa esta decisão
- `INC-PC-2026-05-21-libvirt-virbr0-docker-forward-drop.md` — incidente que motivou a regra do `DOCKER-USER`
- `OCI_2026-05-05_wireguard-mullvad-exit-node-tailscale.md` — referência de padrão de policy routing para uso futuro
