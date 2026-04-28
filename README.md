# The Grimoire
![CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey?style=for-the-badge)

Runbook pessoal de TI: registro dos problemas que resolvi, do que configurei e de como ficou. Escrito para eu mesmo no futuro — mas público, porque boa documentação não tem razão para ficar escondida.

Cada documento é um snapshot do que aconteceu: o contexto, o porquê, o que foi feito e como verificar. Sem padding — só o que é útil.

---

## Como está organizado

Os arquivos seguem o padrão `{ESCOPO}_{DATA}_{slug}.md`:

| Escopo | Ambiente |
|---|---|
| `PC_` | Computador pessoal (Fedora 43, KDE Plasma 6, Wayland) |
| `SRV_` | Servidor VPS (Ubuntu 24.04, Docker, Nextcloud AIO, Nginx Proxy Manager, Tailscale) |
| `NOTE_` | Notebook pessoal (Fedora 43, KDE Plasma 6, Wayland) |

Cada documento começa com o tipo que define sua natureza:

- `Fix:` — resolução de um problema específico, com causa raiz
- `Setup:` — configuração de algo do zero
- `Config:` — decisão de configuração documentada

---

## Convenções e segurança

Valores sensíveis (IPs de servidor, usuários, domínios próprios) nunca aparecem nos documentos — são substituídos por `{{PLACEHOLDERS}}` e mapeados em `values.local`, que está no `.gitignore` e nunca é commitado.

As regras completas de nomenclatura, estrutura de documentos e o sistema de placeholders estão em [GUIA-DOCUMENTACAO.md](GUIA-DOCUMENTACAO.md).

---

## Stack documentada

`Fedora 43` · `KDE Plasma 6` · `Wayland` · `PipeWire` · `systemd` · `Docker` · `Nextcloud AIO` · `Nginx Proxy Manager` · `Tailscale` · `AdGuard Home`

---

## Licença

[CC BY-SA 4.0](LICENSE) — você pode usar, compartilhar e adaptar o conteúdo, desde que dê crédito e mantenha a mesma licença em derivações.
