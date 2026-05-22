# ADR: Rede libvirt isolada da rede Docker no PC pessoal

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, libvirt + Docker convivendo no mesmo host
- **Status:** Aceita

---

## Contexto

O PC pessoal hospeda dois sistemas de virtualização/containerização:

- **libvirt + QEMU/KVM** rodando VMs descartáveis pra teste, experimentação com OS, sandboxes de desenvolvimento. Rede default `virbr0` em `192.168.122.0/24`.
- **Docker** rodando containers de aplicações pessoais (n8n e similares). Bridges default `docker0` em `172.17.0.0/16` e bridges custom em `172.18.0.0/16`.

Surge a decisão de como integrar (ou não) a rede das VMs com a rede dos containers. Três abordagens são tecnicamente viáveis:

1. **Isolamento total** — VMs e containers em redes separadas, sem rota direta entre eles. Comunicação possível só via interfaces externas (host atuando como roteador, ou via internet).
2. **Bridge física compartilhada (`br0`)** — host expõe uma bridge layer 2 onde tanto VMs quanto containers (via macvlan) se conectam, virando hosts de primeira classe na LAN doméstica.
3. **Rota explícita entre subnets** — manter redes separadas mas adicionar regras de roteamento e firewall pra permitir comunicação entre `virbr0` e bridges Docker.

Há também um fator técnico crítico recém-descoberto neste setup: o **conflito de backends de firewall** entre libvirt, Docker e Tailscale quando todos escrevem na mesma tabela `ip filter` (documentado em `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md`). A decisão de isolamento influencia diretamente quanto esse conflito potencialmente cria problemas — quanto mais integração entre redes, mais regras cruzadas, mais oportunidade de conflito.

---

## Decisão

**VMs e containers Docker operam em redes completamente isoladas:**

- VMs usam a rede `default` do libvirt: `virbr0`, subnet `192.168.122.0/24`, modo NAT. Regras de firewall em tabela `ip libvirt_network` (após adoção de `firewall_backend = "nftables"`).
- Containers Docker usam bridges nativas (`docker0`, e bridges custom de compose stacks) em subnets `172.17.x.x` e `172.18.x.x`. Regras de firewall em tabela `ip docker-bridges` (após adoção de `firewall-backend: nftables`).
- **Nenhuma rota direta entre VMs e containers.** Se uma VM precisar conversar com um container, o caminho é via porta exposta pelo Docker em `localhost`/`0.0.0.0` (acessível como `192.168.122.1:PORTA` a partir da VM).
- Cada um sai pra internet via NAT próprio através da `enp4s0`.

Esta decisão é a base do isolamento técnico que permite que `ADR-PC-2026-05-22-nftables-backend-unificado-firewall.md` (backend nftables unificado) funcione de forma limpa: como cada componente tem sua própria tabela e não precisa de regras cruzadas, não há sobreposição.

---

## Alternativas consideradas

### Bridge física compartilhada (`br0`)

Criar uma bridge `br0` no host, mover a interface física `enp4s0` para dentro dela, e conectar VMs e containers (via macvlan) nessa bridge. VMs e containers viram hosts de primeira classe na rede 192.168.1.0/24 (LAN doméstica).

**Vantagens:** comunicação direta L2, IPs reais na LAN, fácil descoberta entre VMs/containers e outros dispositivos da rede (TV, celular, outros computadores).

**Descartada porque:**
- Complexidade de configuração alta (bridge no NetworkManager, ajustes nas duas stacks, mudanças de routing no host)
- Risco de quebrar conectividade do próprio host durante o setup
- VMs e containers passariam a competir por IPs do DHCP da LAN, gerando dependência do roteador doméstico
- Implicações de segurança: containers e VMs ficam expostos diretamente na LAN doméstica, contornando o NAT
- O ganho não compensa o caso de uso atual — VMs são descartáveis e experimentais, não servidores que precisam ser descobertos da LAN

### Rota explícita entre `virbr0` e bridges Docker

Manter redes separadas em tabelas nftables próprias, mas adicionar regras explícitas permitindo forward direto entre `virbr0` e `docker0` (e bridges customizadas).

**Vantagens:** isolamento topológico mantido, comunicação possível só onde necessário.

**Descartada porque:**
- Multiplicaria a superfície de configuração: cada nova rede Docker (cada `docker-compose` que cria uma bridge custom) demandaria nova regra
- Reabriria a porta pro problema do INC: regras cruzadas entre tabelas `libvirt_network` e `docker-bridges` exigiriam policies adicionais no firewalld pra rotear, potencialmente recriando o conflito de backends
- O caso de uso atual não exige comunicação VM↔container — não há serviço Docker que VMs precisem acessar diretamente
- Se eventualmente precisar, basta expor a porta no host e a VM acessa via `192.168.122.1:PORTA`

### Rede libvirt com policy routing dedicada (espelhando padrão OCI)

Criar uma segunda rede libvirt isolada (ex.: `vmnet-vpn`, subnet `192.168.123.0/24`) com policy routing dedicado, similar ao setup OCI com WireGuard/Tailscale documentado em `OCI_2026-05-05_wireguard-mullvad-exit-node-tailscale.md`.

**Considerada mas adiada porque:** não há necessidade atual de VPN nas VMs, e a rede `default` resolve. **Documentado como caminho futuro:** se algum dia surgir necessidade de VMs com saída via VPN (ex.: rodar uma VM pra testar geo-bloqueio ou navegar via Mullvad), criar uma segunda rede libvirt com policy routing dedicado, mantendo a `default` intocada.

---

## Consequências

### Positivas

- **Setup limpo e diagnosticável.** Cada componente em sua própria tabela nftables, sem regras cruzadas, sem chains que dependem de outras chains externas.
- **Reforça o isolamento de backend** (ver `ADR-PC-2026-05-22-nftables-backend-unificado-firewall.md`). A decisão de isolamento de redes e a decisão de backend nftables próprio funcionam em sinergia.
- **Segurança por padrão:** VMs comprometidas não conseguem chegar diretamente nos containers (n8n, etc.) e vice-versa.
- **Manutenção mínima:** novas redes Docker (cada `docker-compose` que cria uma bridge custom) não exigem regras adicionais no libvirt ou nas policies firewalld.
- **Histórico relevante:** a tentativa anterior de "facilitar coexistência" via regras manuais em `DOCKER-USER` foi uma das fontes do problema diagnosticado no INC. Manter as redes isoladas elimina essa necessidade.

### Negativas / Trade-offs

- Se uma VM precisar conversar com um container (ex.: testar uma API exposta pelo n8n), tem que ir pela porta exposta no host. Da VM, o IP `192.168.122.1` (gateway) é o IP do host; portas expostas pelo Docker em `0.0.0.0` são acessíveis nesse endereço.
- Sem descoberta automática entre VMs e containers — cada lado é uma ilha. Mais simples conceitualmente, mas se um dia o caso de uso exigir isso, será necessária nova ADR.
- A separação não impede a coordenação manual: se algo precisa ser visível entre os dois lados, é configurável (porta exposta + IP do host), só não vem "de graça".

### Caminho futuro

Se surgir necessidade de:

- **VMs passarem por VPN** (Mullvad/Proton) — criar uma segunda rede libvirt (`vmnet-vpn`, subnet diferente) com policy routing dedicado, espelhando padrão do OCI WireGuard. A `default` continua intocada para VMs que saem direto pelo host.
- **Comunicação extensa VM↔container** — esta decisão precisa ser revista (nova ADR). O caminho mais limpo seria adicionar uma rede Docker em modo `host` ou expor portas no host explicitamente, em vez de tentar rotear entre as tabelas nftables próprias.

---

## Referências

- `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md` — setup completo que materializa esta decisão
- `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md` — incidente que reforçou a importância do isolamento
- `ADR-PC-2026-05-22-nftables-backend-unificado-firewall.md` — decisão complementar de backend de firewall
- `OCI_2026-05-05_wireguard-mullvad-exit-node-tailscale.md` — referência de padrão de policy routing para uso futuro
