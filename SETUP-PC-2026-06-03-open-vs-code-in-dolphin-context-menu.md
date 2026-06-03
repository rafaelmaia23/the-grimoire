# SETUP: Option `Open in VS Code` in `Dolphin` context menu

- **Data:** 2026-06-03
- **Local:** PC Pessoal
- **Sistema:** Fedora 44, KDE Plasma 6, Wayland

## Contexto

Setup adiciona a opção de abrir com o vs code no menu de contexto do dolphin.

## Configuração

1 - Criar pasta onde o serviço fica:

```bash
mkdir -p ~/.local/share/kio/servicemenus/
```

2 - Criar arquivo do serviço:

```bash
nano ~/.local/share/kio/servicemenus/vscode.desktop
```

conteudo do serviço:

```text
[Desktop Entry]
Type=Service
ServiceTypes=KonqPopupMenu/Plugin
MimeType=all/all;
Actions=openInVSCode;
X-KDE-Priority=TopLevel

[Desktop Action openInVSCode]
Name=Open in VS Code
Icon=vscode
Exec=code %f
```

salvar com `ctrl + o` sair com `ctrl + x`

3 - Tornar executável e recarregar o Dolphin:

```bash
chmod +x ~/.local/share/kio/servicemenus/vscode.desktop
kbuildsycoca6 --noincremental
```
