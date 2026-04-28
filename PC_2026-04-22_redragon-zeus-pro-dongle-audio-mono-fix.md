# Fix: Redragon Zeus Pro (H510-PRO) — Apenas um canal de áudio via dongle no Linux

- **Data:** 2026-04-22
- **Local** Pc pessoal
- **Sistema:** Fedora 43, KDE Plasma, PipeWire 1.4.11, Kernel 6.19.12
- **Hardware:** Redragon Zeus Pro H510-PRO (Weltrend `040b:0897`)

---

## Problema

Com o headset conectado via dongle USB 2.4GHz, apenas o canal esquerdo funcionava. Ajustar o volume pelo painel do KDE Plasma piorava a situação — o canal direito ficava completamente mudo em qualquer volume abaixo de 100%.

Via cabo P2 e via Bluetooth funcionava normalmente nos dois canais.

---

## Causa Raiz

O dongle expõe **dois controles PCM separados** no ALSA:

- `PCM,0` — controle **estéreo** (Front Left + Front Right), que é o controle real do hardware
- `PCM,1` — controle **mono** (índice 1), usado como referência pelo PipeWire

O PipeWire usava o `PCM,1` para controlar o volume e deixava o `PCM,0` permanentemente em 0%. Com `PCM,0` zerado, o canal direito ficava mudo. Em 100% do painel do KDE funcionava porque o PipeWire fazia soft-volume nesse caso, contornando o hardware — mas qualquer valor abaixo disso deixava o canal direito sem sinal.

Confirmação:

```bash
amixer -c 1
# Mostrava:
# Simple mixer control 'PCM',0
#   Front Left: Playback 0 [0%]
#   Front Right: Playback 0 [0%]
#
# Simple mixer control 'PCM',1
#   Mono: Playback 51 [51%]
```

---

## Solução

### 1. Subir o `PCM,0` para 100% nos dois canais

```bash
amixer -c 1 sset 'PCM' 100% 100%
```

### 2. Salvar o estado do mixer para persistir no boot

```bash
sudo alsactl store 1
```

O `alsa-restore.service` do systemd restaura esse estado automaticamente no boot.

### 3. Criar regra udev para restaurar ao reconectar o dongle

Como dispositivos USB são plugados depois do boot, o `alsa-restore` sozinho não é suficiente. A regra abaixo garante que o mixer seja restaurado toda vez que o dongle for conectado.

**Arquivo:** `/etc/udev/rules.d/90-redragon-headset.rules`

```
ACTION=="add", SUBSYSTEM=="sound", ATTRS{idVendor}=="040b", ATTRS{idProduct}=="0897", RUN+="/usr/sbin/alsactl restore 1"
```

Após criar o arquivo:

```bash
sudo udevadm control --reload-rules
```

---

## Estado final

- O `PCM,0` é salvo em 100% FL+FR em `/var/lib/alsa/asound.state`
- A regra udev restaura esse estado ao conectar o dongle
- O PipeWire controla o volume normalmente via soft-volume, sem nunca alterar o `PCM,0`
- Funciona em qualquer nível de volume pelo painel do KDE Plasma
- Funciona após reboot
- Funciona ao reconectar o dongle sem reiniciar

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/udev/rules.d/90-redragon-headset.rules` | Criado | Regra udev para restaurar mixer ao conectar o dongle |
| `/var/lib/alsa/asound.state` | Modificado | Estado do mixer ALSA salvo pelo `alsactl store 1` |

---

## Referência rápida — comandos úteis

```bash
# Ver controles ALSA do dongle
amixer -c 1

# Verificar se PCM,0 está em 100% (deve estar sempre assim)
amixer -c 1 get 'PCM'

# Corrigir manualmente se necessário (não deveria precisar)
amixer -c 1 sset 'PCM' 100% 100%
sudo alsactl store 1

# Ver sinks do PipeWire
pactl list sinks short

# Ver card do headset
pactl list cards | grep -A 10 "H510"
```
