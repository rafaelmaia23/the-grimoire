# ADR: Backend nftables nativo unificado para libvirt, Docker e firewalld

- **Data:** 2026-05-22
- **Local:** PC pessoal
- **Sistema:** Fedora 44, libvirt + Docker + Tailscale + firewalld
- **Status:** Aceita

---

## Contexto

No Fedora 44, o backend padrão de firewall é o nftables nativo. Porém, vários componentes que mexem em regras de firewall ainda usam por default o `iptables-nft` — uma camada de compatibilidade que traduz comandos `iptables` em regras nftables. Isso permite que esses componentes funcionem em qualquer distro, mas tem um efeito colateral importante: todos eles escrevem na **mesma tabela** `ip filter` (e `ip nat`), compartilhando o espaço de nomes.

No PC pessoal, os seguintes componentes coexistem:

| Componente | Backend padrão | Tabela onde escreve por default |
|---|---|---|
| firewalld | nftables nativo (Fedora 44) | `inet firewalld` |
| libvirt | iptables-nft | `ip filter`, `ip nat` |
| Docker | iptables-nft (default `iptables: true`) | `ip filter`, `ip nat` |
| Tailscale | iptables-nft (não configurável) | `ip filter`, `ip nat` |

Quando libvirt, Docker e Tailscale escrevem todos em `ip filter` ao mesmo tempo, surgem situações em que:

- Chains pulam pra chains de outros componentes em ordens imprevisíveis
- Policy `DROP` definida por um afeta tráfego do outro
- Counters sobem mas pacotes morrem em chains que nem `iptables -L` mostra
- Regras manuais (`iptables -I DOCKER-USER ...`) funcionam às vezes e não funcionam outras, dependendo do estado momentâneo de cada componente

Este conflito **foi a causa raiz** do incidente extenso documentado em `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md`, onde várias horas foram gastas tentando aplicar fixes paliativos antes que a sobreposição de backends fosse identificada como causa raiz.

---

## Decisão

**Forçar libvirt e Docker a usar nftables nativo** em vez do default `iptables-nft`. Tailscale fica como está (não é configurável, mas processa em chains próprias `ts-*`).

Configurações aplicadas:

### libvirt
Em `/etc/libvirt/network.conf`:
```
firewall_backend = "nftables"
```

Resultado: libvirt cria sua própria tabela `ip libvirt_network` (e `ip6 libvirt_network`), com chains `forward`, `guest_input`, `guest_output`, `guest_cross`, `guest_nat`. Todas as regras de NAT e forwarding da rede default ficam nessa tabela isolada.

### Docker
Em `/etc/docker/daemon.json`:
```json
{
  "firewall-backend": "nftables"
}
```

Resultado: Docker cria sua própria tabela `ip docker-bridges` (e `ip6 docker-bridges`), com chains de filter, NAT e maps por bridge. Todas as regras dos containers ficam nessa tabela isolada.

### firewalld
Em `/etc/firewalld/firewalld.conf`:
```
FirewallBackend=nftables
```

(Default no Fedora 44, mantido.)

### Tailscale
Mantido em iptables-nft. Continua escrevendo em `ip filter` e `ip nat`, mas agora **sozinho** nessas tabelas — sem disputa com libvirt ou Docker.

---

## Alternativas consideradas

### Manter todos em iptables-nft (status quo)

Foi a configuração inicial. **Não funcionou** pelo motivo descrito no contexto: sobreposição de regras na tabela `ip filter` quando múltiplos componentes escrevem nela.

**Descartada porque** já estava demonstrado, pela investigação extensa do dia 22, que esse modo causa instabilidade. Mesmo aplicando paliativos (regras no `DOCKER-USER`, policies cross-zone no firewalld, masquerade na zona libvirt), o problema voltava em condições aparentemente aleatórias.

### Desabilitar firewalld totalmente

Algumas distros simplificam permitindo só iptables ou só nftables raw, sem firewalld. Reduziria a complexidade pra 3 atores (libvirt + Docker + Tailscale).

**Descartada porque** firewalld é parte do Fedora padrão, fornece zonas e policies úteis pra gestão de zonas confiáveis (FedoraWorkstation), e desabilitá-lo geraria divergência grande em relação ao Fedora upstream — dificultando troubleshooting com docs oficiais e exigindo recriar do zero as regras de proteção do host.

### Desinstalar Tailscale e usar outra solução de VPN

Tailscale é a fonte de uma das três stacks que disputavam a `ip filter`. Sem ele, restavam só Docker + libvirt, mais fáceis de coordenar.

**Descartada porque** Tailscale é crítico no setup: dá acesso à VPS Hostinger, à VM OCI (homelab), AdGuard Home como DNS, app mobile como exit node. Substituir por WireGuard nativo seria possível mas exigiria reconstruir todo o mesh, e a raiz do problema (sobreposição de regras) continuaria possível com qualquer outra solução que use `iptables-nft`.

### Migrar tudo pra iptables-legacy (em vez de iptables-nft)

Voltar pra `update-alternatives --set iptables /usr/sbin/iptables-legacy`. iptables-legacy escreve em tabela kernel diferente do nftables, sem conflito.

**Descartada porque** iptables-legacy está deprecado nas distros modernas, conforme noticiado pela própria documentação Docker e em consenso da comunidade. Investir em algo que vai sumir não faz sentido. Direção correta é avançar pra nftables nativo, não recuar.

---

## Consequências

### Positivas

- **Cada componente em sua própria tabela nftables.** Sem sobreposição, sem ordem imprevisível, sem regras invisíveis a `iptables -L`.
- **Diagnóstico mais simples.** `sudo nft list tables` mostra exatamente quem está rodando o quê. Counters de cada tabela são independentes.
- **Resistência a regressão.** Se um componente reiniciar e recriar suas chains, ele recria na própria tabela — não afeta os outros.
- **Alinhamento com direção do upstream.** Docker já anunciou que `nftables` vai ser o backend default em versão futura, e iptables será deprecado. libvirt já tem suporte nativo. Estar nesse caminho agora é estar na frente.

### Negativas / Trade-offs

- **Suporte a nftables no Docker ainda é experimental** (até Docker 29). Possibilidade de bugs específicos do backend nftables aparecerem em features avançadas (Docker Swarm, redes overlay complexas). Mitigação: caso pessoal não usa essas features, e voltar pra `"firewall-backend": "iptables"` é uma linha de mudança no `daemon.json`.
- **Migração de regras `DOCKER-USER` precisaria de cuidado.** Quem tem regras customizadas em `DOCKER-USER` (chain iptables tradicional) precisa migrar pra chain equivalente da tabela `docker-bridges` no nftables, que tem estrutura diferente. No setup atual, **nenhuma regra customizada é necessária** (cada componente cuida do seu NAT), então isso não é problema agora.
- **Documentação ainda escassa pra debugging.** Diagnosticar problemas em `docker-bridges` exige conhecer a estrutura nftables nativa do Docker (maps, jumps por interface), em vez do iptables tradicional que tem mais material disponível. Mitigação: documentar no grimoire (ver `REF-PC-2026-05-22-diagnostico-rede-nftables-fedora.md`).

### Caminho futuro

Quando o `nftables` virar default oficial no Docker (anunciado pra release futura) e sair do estado "experimental", esta ADR pode ser revogada e a configuração `firewall-backend: nftables` passa a ser implícita. Reavaliar em cada major release do Docker.

Para libvirt, a configuração `firewall_backend = "nftables"` é estável e pode permanecer indefinidamente. É o caminho recomendado upstream pra distros que usam nftables como default.

---

## Referências

- `INC-PC-2026-05-22-libvirt-docker-tailscale-firewall-backend-conflict.md` — incidente que motivou esta decisão
- `SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md` — setup que aplica esta decisão na prática
- `ADR-PC-2026-05-22-libvirt-rede-isolada-do-docker.md` — decisão complementar sobre isolamento de redes
- Documentação Docker sobre nftables: https://docs.docker.com/engine/network/firewall-nftables/
- Documentação libvirt sobre firewall_backend: https://libvirt.org/firewall.html
- Documentação firewalld sobre FirewallBackend: https://firewalld.org/documentation/configuration/main-conf.html
