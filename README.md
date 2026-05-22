# The Grimoire
![CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey?style=for-the-badge)

Runbook pessoal de TI: registro dos problemas que resolvi, do que configurei, das decisões que tomei e de como ficou. Escrito para eu mesmo no futuro — mas público, porque boa documentação não tem razão para ficar escondida.

Cada documento é um snapshot do que aconteceu: o contexto, o porquê, o que foi feito e como verificar. Sem padding — só o que é útil.

---

## Como está organizado

Os arquivos seguem o padrão `{TIPO}-{ESCOPO}-{YYYY-MM-DD}-{slug}.md`.

### Tipos

| Tipo | Propósito |
|---|---|
| `SETUP` | Instalação/configuração de algo do zero, reproduzível |
| `FIX` | Resolução enxuta de problema pontual |
| `INC` | Incidente em profundidade com timeline e causa raiz |
| `ADR` | Decisão arquitetural — por que escolhi X e não Y |
| `RUNBOOK` | Procedimento operacional recorrente |
| `REF` | Referência/cheat sheet de consulta rápida |

### Escopos

| Escopo | Ambiente |
|---|---|
| `PC` | Computador pessoal (Fedora 44, KDE Plasma 6, Wayland) |
| `NOTE` | Notebook pessoal (Fedora, KDE Plasma 6, Wayland) |
| `SRV` | Servidor VPS Hostinger (Ubuntu 24.04, Docker, Nextcloud AIO, Nginx Proxy Manager, Tailscale) |
| `OCI` | VM Oracle Cloud Infrastructure (Ubuntu 24.04 ARM, Tailscale, WireGuard) |

> **Nota:** Documentos anteriores a 2026-05-22 ainda usam o padrão antigo `{ESCOPO}_{DATA}_{slug}.md`. A migração para o novo padrão está em andamento (ver `TODO-MIGRACAO.md`).

---

## Convenções e segurança

Valores sensíveis (IPs de servidor, usuários, domínios próprios) nunca aparecem nos documentos — são substituídos por `{{PLACEHOLDERS}}` e mapeados em `values.local`, que está no `.gitignore` e nunca é commitado.

As regras completas de nomenclatura, estrutura de documentos e o sistema de placeholders estão em [GUIA-DOCUMENTACAO.md](GUIA-DOCUMENTACAO.md).

---

## Stack documentada

`Fedora 44` · `KDE Plasma 6` · `Wayland` · `PipeWire` · `systemd` · `Docker` · `QEMU/KVM` · `libvirt` · `Nextcloud AIO` · `Nginx Proxy Manager` · `Tailscale` · `WireGuard` · `AdGuard Home`

---

## Licença

[CC BY-SA 4.0](LICENSE) — você pode usar, compartilhar e adaptar o conteúdo, desde que dê crédito e mantenha a mesma licença em derivações.
