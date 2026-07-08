# Fix: Redragon Zeus Pro (H510-PRO) — Áudio via dongle mudo abaixo de 100% no Linux

- **Data (revisão):** 2026-07-08
- **Data (original):** 2026-04-22
- **Local:** Pc pessoal
- **Sistema:** Fedora 44, KDE Plasma 6 (Wayland), PipeWire 1.6.7 / WirePlumber
- **Hardware:** Redragon Zeus Pro H510-PRO (Weltrend `040b:0897`), dongle 2.4GHz

---

## Resumo (leia isto primeiro)

O dongle 2.4GHz do H510-PRO tem um mixer de hardware defeituoso: o controle `PCM,0`
só tem estados úteis 0 ou ~99 (sem passos intermediários) e a curva de dB reportada
é estreita demais. Quando o PipeWire usa esse controle para volume, qualquer nível
abaixo do limiar colapsa para silêncio.

**Solução definitiva (2026-07-08):** forçar o WirePlumber a NÃO usar o mixer de
hardware desse card (`api.alsa.soft-mixer = true`) e manter o `PCM,0` pinado em 100%.
Assim todo o volume passa a ser feito em software e o hardware fica sempre escancarado.

> Bluetooth e cabo P2 **nunca** foram afetados — eles não passam pelo card USB do dongle
> (BT vai pelo BlueZ; o P2 vai pelo ALC897 onboard). O bug é exclusivo do card USB do dongle.

---

## Histórico do problema

### Sintoma original (abril/2026)
Via dongle, apenas o canal **esquerdo** funcionava. Abaixar o volume no painel do KDE
piorava: o canal direito mutava completamente abaixo de 100%. Via cabo e Bluetooth, ok.

### Recorrência (julho/2026, após att do Plasma/sistema)
O áudio via dongle parou por completo em qualquer volume. Dois fatores se somaram:

1. **`PCM,0` voltou a zerar** — o comportamento de base do bug.
2. **A ordem de enumeração dos cards mudou na att:**
   - Antes: fone = `card 1`, GPU HDMI = `card 0`
   - Depois: fone = `card 0`, GPU HDMI = `card 1`
   A regra udig antiga tinha `alsactl restore 1` **chumbado no índice**, então passou
   a restaurar o card errado (o HDMI) ao conectar o dongle. O `PCM,0` do fone nunca
   mais era restaurado. **Lição: índice de card USB não é confiável; nunca chumbar
   número — referenciar por nome.**

Durante o diagnóstico de julho, ao aplicar só o `soft-mixer` (sem pinar o `PCM,0`),
o áudio ficou mudo em 100% também — porque o soft-mixer também impede o PipeWire de
**subir** o `PCM,0`, e ele estava em 0. Isso confirmou que a solução exige as **duas**
peças juntas.

---

## Causa raiz

O card USB do dongle expõe um controle de volume de hardware (`PCM`) com dois problemas:

- **Faixa binária:** salta de 0 direto para ~99, sem valores intermediários.
- **Curva de dB estreita/errada:** o mapeamento de dB do PipeWire colapsa qualquer
  volume mais baixo para silêncio.

Quando o WirePlumber usa o mixer de hardware para controlar volume (comportamento
padrão quando o hardware "suporta" controle de volume), abaixar o volume zera o
`PCM,0` e corta o áudio. Atualizações do sistema periodicamente restauram o
comportamento padrão e/ou reordenam os cards, re-quebrando qualquer fix baseado em
pinagem + índice.

---

## Solução definitiva

Duas peças complementares:

### 1. Forçar volume por software (WirePlumber ignora o mixer de hardware)

**Arquivo:** `~/.config/wireplumber/wireplumber.conf.d/51-h510-soft-mixer.conf`

```
monitor.alsa.rules = [
  {
    matches = [
      {
        device.name = "alsa_card.usb-XiiSound_Technology_Corporation_H510-PRO_Wireless_headset-00"
      }
    ]
    actions = {
      update-props = {
        api.alsa.soft-mixer = true
      }
    }
  }
]
```

Aplicar:

```bash
systemctl --user restart wireplumber
```

> `api.alsa.soft-mixer = true` desabilita o mixer de hardware para volume/mute; todo o
> volume passa a ser feito em software, deixando o mixer de hardware intocado.
> Se algum dia o soft-mixer sozinho não bastar, adicionar `api.alsa.ignore-dB = true`
> na mesma seção `update-props` (ignora a curva de dB errada do driver).

### 2. Pinar o `PCM,0` do hardware em 100% (para o sinal sempre passar cru)

```bash
# Sobe os dois canais para 100% (usar o NOME do card, não o índice)
amixer -c headset sset 'PCM' 100% 100%

# Verificar (deve mostrar Front Left e Front Right em 100%)
amixer -c headset get 'PCM'

# Persistir o estado do mixer
sudo alsactl store
```

### 3. Regra udev — restaurar o mixer ao reconectar o dongle (POR NOME)

Como o dongle é plugado depois do boot, o `alsa-restore.service` sozinho não basta.
A regra restaura o estado a cada conexão. **Referência pelo nome do card (`headset`),
não pelo índice**, para sobreviver a reordenações.

**Arquivo:** `/etc/udev/rules.d/90-redragon-headset.rules`

```
ACTION=="add", SUBSYSTEM=="sound", ATTRS{idVendor}=="040b", ATTRS{idProduct}=="0897", RUN+="/usr/sbin/alsactl restore headset"
```

Aplicar:

```bash
sudo udevadm control --reload-rules
```

---

## Estado final

- WirePlumber faz todo o volume em software; **nunca** toca no `PCM,0`.
- `PCM,0` fica pinado em 100% FL+FR, salvo em `/var/lib/alsa/asound.state`.
- A regra udev restaura o `PCM,0` por **nome** ao conectar o dongle (imune a reordenação).
- Volume funciona em qualquer nível pelo painel do KDE, nos dois canais.
- Sobrevive a reboot e a reconexão do dongle sem reiniciar.

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `~/.config/wireplumber/wireplumber.conf.d/51-h510-soft-mixer.conf` | Criado | Força volume por software no card do dongle (`api.alsa.soft-mixer`) |
| `/etc/udev/rules.d/90-redragon-headset.rules` | Modificado | Restaura o mixer ao conectar o dongle — agora por NOME (`restore headset`) |
| `/var/lib/alsa/asound.state` | Modificado | Estado do mixer ALSA salvo pelo `alsactl store` |

---

## Diagnóstico — como confirmar que é este bug

```bash
# É só o dongle? (BT e cabo funcionam → sim, é o card USB)
# Identificar os cards e qual é o do dongle:
aplay -l                         # shortname do card (ex.: card 0: headset)
pactl list cards short           # device.name do PipeWire
wpctl status                     # sinks/devices ativos

# Ver o controle de hardware do dongle (o vilão):
amixer -c headset get 'PCM'      # se estiver em 0% e o som cortar abaixo de 100% → é este bug

# Ver eventos USB ao replugar o dongle:
sudo dmesg -w                    # (precisa de sudo)
lsusb                            # dongle = 040b:0897 Weltrend / BT = 2357:0604 TP-Link
```

---

## Referência rápida — comandos úteis

```bash
# Verificar o PCM de hardware (deve estar sempre 100% FL+FR)
amixer -c headset get 'PCM'

# Corrigir manualmente se necessário (não deveria precisar)
amixer -c headset sset 'PCM' 100% 100%
sudo alsactl store

# Reaplicar config do WirePlumber
systemctl --user restart wireplumber

# Sinks do PipeWire / card do headset
pactl list sinks short
pactl list cards | grep -A 10 "H510"
```

---

## Notas / lições

- **Nunca chumbar índice de card USB** em regra udev ou script — a numeração muda entre
  atualizações e reboots. Usar sempre o nome do card ou o `device.name`.
- A abordagem de abril (só pinar `PCM,0` + udev por índice) era frágil: dependia da
  sorte de o PipeWire escolher outro controle e re-quebrava em toda att. O soft-mixer
  torna a solução determinística.
- `soft-mixer` sozinho **não basta** neste hardware: ele impede o PipeWire de rebaixar
  o `PCM,0`, mas também de subi-lo. Sem o pin em 100%, o resultado é mudo total.
```
