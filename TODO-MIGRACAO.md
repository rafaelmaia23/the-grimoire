# TODO: Migração dos arquivos antigos para o novo padrão

- **Data:** 2026-05-22
- **Status:** Pendente — migração gradual

---

## Contexto

O guia de documentação foi atualizado em 2026-05-22 com novo padrão de nomenclatura:

- **Antigo:** `{ESCOPO}_{DATA}_{slug}.md` com tipo no `# H1` interno (`Fix:`, `Setup:`, `Config:`)
- **Novo:** `{TIPO}-{ESCOPO}-{DATA}-{slug}.md` com hífen como separador único, e taxonomia expandida (`SETUP`, `FIX`, `INC`, `ADR`, `RUNBOOK`, `REF`)

Documentos criados a partir de 2026-05-22 já seguem o novo padrão. Documentos anteriores devem ser renomeados e reclassificados gradualmente.

---

## Como fazer a migração de cada arquivo

Para cada arquivo da tabela abaixo:

1. **Renomear** o arquivo conforme a coluna "Novo nome"
2. **Editar o `# H1`** trocando `Fix:` / `Setup:` / `Config:` pelo tipo novo em maiúsculas (`FIX:` / `SETUP:` / `INC:` / `REF:` etc.)
3. **Conferir o cabeçalho** (Data, Local, Sistema) — geralmente já está ok
4. Se a tipagem mudou (ex.: `Config:` virou `REF:`), pode ser necessário **reorganizar seções** para se alinhar com o que se espera do novo tipo (ver guia)
5. Commit com mensagem descritiva: `docs: migra <nome-antigo> para novo padrão como <TIPO>`

Não fazer tudo de uma vez — migrar 1-2 por commit para manter histórico legível.

---

## Tabela de migração

| Nome atual | Tipo atual no H1 | Novo tipo proposto | Justificativa | Novo nome |
|---|---|---|---|---|
| `PC_2026-04-22_redragon-zeus-pro-dongle-audio-mono-fix.md` | `Fix:` | `FIX` | Problema pontual, causa clara, fix direto. Não tem timeline complexa. | `FIX-PC-2026-04-22-redragon-zeus-pro-dongle-audio-mono.md` |
| `PC_2026-04-23_remote-toggle-script-acesso-remoto.md` | `Setup:` | `SETUP` | Instalação de script com configuração persistente. | `SETUP-PC-2026-04-23-remote-toggle-script-acesso-remoto.md` |
| `PC_2026-04-23_rustdesk-flatpak-to-rpm-fedora43-wayland.md` | `Setup:` | `SETUP` | Migração/configuração de software com passos reproduzíveis. | `SETUP-PC-2026-04-23-rustdesk-flatpak-to-rpm-fedora43-wayland.md` |
| `PC_2026-04-28_bash-histcontrol-ignorespace.md` | `Config:` | `REF` | É consultivo, curto, sobre como usar uma feature do bash. Não tem "passos de instalação" significativos — só um export no `.bashrc`. | `REF-PC-2026-04-28-bash-histcontrol-ignorespace.md` |
| `PC_2026-04-28_ssh-keys-restauracao-notebook-antigo.md` | `Setup:` | `SETUP` | Procedimento de restauração de configuração. | `SETUP-PC-2026-04-28-ssh-keys-restauracao-notebook-antigo.md` |
| `PC_2026-04-28_template-markdown-xdg.md` | `Setup:` | `SETUP` | Setup de um template XDG, passos reproduzíveis. | `SETUP-PC-2026-04-28-template-markdown-xdg.md` |
| `SRV_2026-04-23_nextcloud-aio-custom-config-trusted-proxy.md` | `Config:` | `SETUP` | Apesar de chamado "Config", tem instalação, arquitetura completa, comandos de aplicação — é setup de algo que precisa ser refeito em reinstalação. | `SETUP-SRV-2026-04-23-nextcloud-aio-custom-config-trusted-proxy.md` |
| `SRV_2026-04-23_nextcloud-davx5-429-trusted-proxy-fix.md` | `Fix:` | `INC` | Tem timeline detalhada (4 incidentes), investigação profunda, causa raiz aprofundada, lições. É INC clássico. | `INC-SRV-2026-04-23-nextcloud-davx5-429-trusted-proxy.md` |
| `SRV_2026-04-24_nextcloud-aio-docker-dns-fix.md` | `Fix:` | `FIX` | Causa direta (DNS upstream faltando no daemon.json), solução enxuta, sem timeline. | `FIX-SRV-2026-04-24-nextcloud-aio-docker-dns.md` |
| `OCI_2026-05-04_wireguard-proton-exit-node-tailscale.md` | `Setup:` | `SETUP` | Setup completo de WireGuard com policy routing. | `SETUP-OCI-2026-05-04-wireguard-proton-exit-node-tailscale.md` |
| `OCI_2026-05-05_wireguard-mullvad-exit-node-tailscale.md` | `Setup:` | `SETUP` | Setup completo, contém também a justificativa Mullvad vs Proton (poderia virar ADR separado no futuro). | `SETUP-OCI-2026-05-05-wireguard-mullvad-exit-node-tailscale.md` |

---

## Observações pós-migração

### Possíveis ADRs a extrair

Os arquivos de WireGuard OCI contêm decisões arquiteturais embutidas (por que Mullvad e não Proton, por que `iif tailscale0` e não `from`, etc.). Em algum momento vale considerar **extrair essas decisões em ADRs separados**, deixando o SETUP focado no procedimento. Sugestões:

- `ADR-OCI-2026-05-05-mullvad-em-vez-de-proton-vpn.md` — registra a decisão Mullvad vs Proton com a tabela de latência
- `ADR-OCI-2026-05-04-policy-routing-iif-em-vez-de-from.md` — registra a decisão de usar `iif tailscale0` em vez de `from 100.64.0.0/10`

Isso fica como **fase 2** da migração — primeiro renomear, depois (talvez) extrair ADRs.

### Atualização das referências cruzadas

Após renomear, atualizar referências internas entre documentos. Lista de cross-references conhecidas a corrigir:

- `SRV_..._nextcloud-davx5-429...` referencia `SRV_..._nextcloud-aio-custom-config...` — atualizar pro novo nome
- `SRV_..._nextcloud-aio-custom-config...` referencia `SRV_..._nextcloud-davx5-429...` — idem

Procurar referências assim:

```bash
grep -rn "PC_\|SRV_\|NOTE_\|OCI_" *.md
```

### Atualização do README.md

O `README.md` também menciona o padrão antigo na seção "Como está organizado". Atualizar após concluir migração de tudo (ou em paralelo, indicando que migração está em andamento).

---

## Checklist global da migração

- [ ] Renomear `PC_2026-04-22_redragon-zeus-pro-dongle-audio-mono-fix.md`
- [ ] Renomear `PC_2026-04-23_remote-toggle-script-acesso-remoto.md`
- [ ] Renomear `PC_2026-04-23_rustdesk-flatpak-to-rpm-fedora43-wayland.md`
- [ ] Renomear `PC_2026-04-28_bash-histcontrol-ignorespace.md` (e reclassificar pra REF)
- [ ] Renomear `PC_2026-04-28_ssh-keys-restauracao-notebook-antigo.md`
- [ ] Renomear `PC_2026-04-28_template-markdown-xdg.md`
- [ ] Renomear `SRV_2026-04-23_nextcloud-aio-custom-config-trusted-proxy.md` (e reclassificar Config → SETUP)
- [ ] Renomear `SRV_2026-04-23_nextcloud-davx5-429-trusted-proxy-fix.md` (e reclassificar Fix → INC)
- [ ] Renomear `SRV_2026-04-24_nextcloud-aio-docker-dns-fix.md`
- [ ] Renomear `OCI_2026-05-04_wireguard-proton-exit-node-tailscale.md`
- [ ] Renomear `OCI_2026-05-05_wireguard-mullvad-exit-node-tailscale.md`
- [ ] Atualizar referências cruzadas entre os documentos renomeados
- [ ] Atualizar `README.md` se necessário
- [ ] (Opcional, fase 2) Extrair ADRs separados dos SETUPs de WireGuard
- [ ] Deletar este `TODO-MIGRACAO.md` após conclusão
