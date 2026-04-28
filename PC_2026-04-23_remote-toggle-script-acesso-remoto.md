# Setup: remote-toggle — Script de Toggle para Acesso Remoto (Sleep + Lock Screen)

- **Data:** 2026-04-23
- **Local:** PC pessoal
- **Sistema:** Fedora 43, KDE Plasma 6.x, Wayland

---

## Contexto

Para acesso remoto via RustDesk funcionar quando ausente, é necessário impedir que o PC entre em sleep e desabilitar o lock screen automático. O objetivo é ter um toggle fácil — ligar antes de sair, desligar ao voltar — sem alterar as configurações permanentes do sistema.

---

## Comportamento do script

**Ao ativar:**
- Bloqueia sleep e idle via `systemd-inhibit`
- Desabilita o lock screen automático via D-Bus (`org.freedesktop.ScreenSaver`)
- Exibe notificação visual e mensagem no terminal

**Ao desativar (rodar de novo):**
- Mata o processo de inibição do sleep
- Libera o inibidor do lock screen
- Tudo volta ao comportamento normal das configurações de energia

---

## Instalação

```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/remote-toggle
```

Conteúdo do script:

```bash
#!/bin/bash

PIDFILE="/tmp/remote-inhibit.pid"
LOCK_COOKIE_FILE="/tmp/remote-lock-cookie"

ligar() {
    systemd-inhibit \
        --what=sleep:idle \
        --who="RemoteAccess" \
        --why="Acesso remoto ativo" \
        sleep infinity &
    echo $! > "$PIDFILE"

    COOKIE=$(qdbus6 org.freedesktop.ScreenSaver \
        /ScreenSaver \
        org.freedesktop.ScreenSaver.Inhibit \
        "RemoteAccess" "Acesso remoto ativo" 2>/dev/null)
    echo "$COOKIE" > "$LOCK_COOKIE_FILE"

    notify-send "☕ Acesso remoto ATIVO" \
        "Sleep e lock screen desabilitados.\nRode remote-toggle para desligar." \
        --icon=network-connect

    echo "✅ Modo remoto ativado — sleep e lock screen desabilitados"
}

desligar() {
    if [ -f "$PIDFILE" ]; then
        PID=$(cat "$PIDFILE")
        kill "$PID" 2>/dev/null
        rm -f "$PIDFILE"
    fi

    if [ -f "$LOCK_COOKIE_FILE" ]; then
        COOKIE=$(cat "$LOCK_COOKIE_FILE")
        qdbus6 org.freedesktop.ScreenSaver \
            /ScreenSaver \
            org.freedesktop.ScreenSaver.UnInhibit \
            "$COOKIE" 2>/dev/null
        rm -f "$LOCK_COOKIE_FILE"
    fi

    notify-send "🔒 Acesso remoto DESATIVADO" \
        "Sleep e lock screen voltaram ao normal." \
        --icon=network-disconnect

    echo "🔒 Modo remoto desativado — sleep e lock screen normais"
}

if [ -f "$PIDFILE" ] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null; then
    desligar
else
    ligar
fi
```

Tornar executável:

```bash
chmod +x ~/.local/bin/remote-toggle
```

---

## Uso

```bash
remote-toggle   # ativa (antes de sair de casa)
remote-toggle   # desativa (ao voltar)
```

O script detecta automaticamente o estado atual pelo PID salvo em `/tmp/remote-inhibit.pid`.

---

## Fluxo recomendado ao sair de casa

1. `remote-toggle` — ativa, sleep e lock desabilitados
2. Sair — acessar via RustDesk de qualquer lugar
3. Ao voltar: `remote-toggle` — desativa, tudo volta ao normal

---

## Atalho de teclado no KDE (opcional)

**System Settings → Shortcuts → Custom Shortcuts → Edit → New → Global Shortcut → Command/URL**

- Nome: `Remote Toggle`
- Comando: `remote-toggle`
- Tecla sugerida: `Meta+R`

---

## Como funciona internamente

| Mecanismo | O que faz |
|---|---|
| `systemd-inhibit --what=sleep:idle` | Impede sleep e suspend via systemd enquanto o processo filho (`sleep infinity`) estiver rodando |
| `org.freedesktop.ScreenSaver.Inhibit` via D-Bus | Registra um inibidor no KDE para desabilitar o lock screen automático; retorna um cookie para desfazer depois |
| `org.freedesktop.ScreenSaver.UnInhibit` | Libera o inibidor usando o cookie salvo |
| PID em `/tmp/remote-inhibit.pid` | Usado para detectar se o modo está ativo e para matar o processo na desativação |

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `~/.local/bin/remote-toggle` | Criado | Script principal de toggle |
| `/tmp/remote-inhibit.pid` | Temporário (runtime) | PID do processo de inibição ativo |
| `/tmp/remote-lock-cookie` | Temporário (runtime) | Cookie D-Bus do inibidor do lock screen |
