# Setup: RustDesk — Migração Flatpak → RPM Nativo + Acesso Remoto no Fedora 43 KDE Wayland

- **Data:** 2026-04-23
- **Local:** PC pessoal
- **Sistema:** Fedora 43, KDE Plasma 6.x, Wayland
- **RustDesk:** 1.4.6 (RPM nativo)

---

## Contexto

O RustDesk estava instalado via Flatpak (`com.rustdesk.RustDesk`). O Flatpak tem integração limitada com o XDG Desktop Portal do KDE no Wayland, causando problemas de captura de tela e sessões não persistentes. A solução é usar o pacote RPM nativo, que se comunica corretamente com o portal.

---

## Problema com o Flatpak

- Sessão de screen sharing expira e o RustDesk não consegue re-requisitar permissão automaticamente
- Popup de autorização do KDE pode não aparecer
- Token de sessão persistente não é salvo corretamente
- Erros comuns: `Failed to create capturer for display 0`, `Failed to get capturer display info`

---

## Migração

### 1. Remover o Flatpak

```bash
flatpak remove com.rustdesk.RustDesk
sudo flatpak remove com.rustdesk.RustDesk  # caso instalado como sistema
rm -rf ~/.var/app/com.rustdesk.RustDesk    # limpar dados antigos
```

### 2. Baixar e instalar o RPM nativo

```bash
cd ~/Downloads
wget https://github.com/rustdesk/rustdesk/releases/download/1.4.6/rustdesk-1.4.6-0.x86_64.rpm
sudo dnf install ./rustdesk-1.4.6-0.x86_64.rpm
```

> **Nota:** Fedora 43 usa DNF5. O comando `localinstall` foi removido — usar `dnf install` direto com o caminho do arquivo.

### 3. Ativar o serviço de sistema

```bash
sudo systemctl enable --now rustdesk
sudo systemctl status rustdesk  # confirmar que está ativo
```

O serviço garante que o RustDesk aceite conexões mesmo sem o app aberto na interface gráfica.

---

## Configuração do Portal XDG (primeira conexão)

Na primeira conexão recebida, o KDE exibe um popup pedindo permissão para compartilhar a tela. É necessário:

1. Clicar em **Permitir**
2. Marcar **"Restaurar em sessões futuras"** (persist mode)

Isso salva um token em `~/.config/rustdesk/RustDesk_local.toml`. Confirmar que foi salvo:

```bash
cat ~/.config/rustdesk/RustDesk_local.toml | grep -i wayland
# Deve aparecer algo com wayland-restore-token
```

Se o popup não aparecer na primeira tentativa, reiniciar o portal e tentar novamente:

```bash
systemctl --user restart xdg-desktop-portal plasma-xdg-desktop-portal-kde
```

As permissões salvas podem ser gerenciadas em:
**System Settings → Privacy & Security → Permissions**

---

## Comportamento esperado após a configuração

- Conexões subsequentes não precisam de novo popup de autorização
- O serviço `rustdesk` aceita conexões no boot automaticamente
- A sessão de screen sharing é restaurada pelo token salvo

---

## Troubleshooting

```bash
# Se a conexão falhar após reconectar (sessão expirada):
systemctl --user restart xdg-desktop-portal plasma-xdg-desktop-portal-kde
# Tentar conectar novamente — deve funcionar sem novo popup

# Ver log do serviço:
journalctl -u rustdesk -f

# Verificar se o portal está rodando:
systemctl --user status xdg-desktop-portal
systemctl --user status plasma-xdg-desktop-portal-kde
```

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `~/.config/rustdesk/RustDesk_local.toml` | Criado pelo app | Token de sessão persistente do Wayland |
| `/etc/systemd/system/rustdesk.service` | Criado pelo instalador | Serviço de sistema do RustDesk |

---

## Referência rápida

```bash
# Status do serviço
sudo systemctl status rustdesk

# Reiniciar se necessário
sudo systemctl restart rustdesk

# Reiniciar portal XDG (fix para sessão expirada)
systemctl --user restart xdg-desktop-portal plasma-xdg-desktop-portal-kde

# Ver token Wayland salvo
cat ~/.config/rustdesk/RustDesk_local.toml | grep -i wayland
```
