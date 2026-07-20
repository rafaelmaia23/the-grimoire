# Fix: Redragon Zeus Pro (H510-PRO) — Áudio via dongle mudo abaixo de 100% no Linux

- **Data (revisão):** 2026-07-20
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

**Solução definitiva (2026-07-20):** forçar o WirePlumber a NÃO usar o mixer de
hardware desse card (`api.alsa.soft-mixer = true`) e, em toda conexão do dongle,
uma regra udev chama `amixer` direto para chumbar `PCM=100%` (valor codificado
na regra — **não** depende do `/var/lib/alsa/asound.state`). Assim todo o volume
passa a ser feito em software e o hardware fica sempre escancarado, e o fix é
imune a corrupção do estado ALSA salvo.

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

### Segunda recorrência (2026-07-20, após nova att do sistema)
Áudio via dongle mudo de novo em qualquer volume. Diagnóstico:

- `soft-mixer = true` continuava aplicado corretamente ao card (confirmado via `pw-dump`).
- `amixer -c headset get PCM` mostrava `Front Left/Right: [0%]` — **o `PCM,0` estava zerado outra vez**.
- A regra udev de julho (`alsactl restore headset`) **funcionou** — mas restaurou zeros.
- Inspecionando `/var/lib/alsa/asound.state`, o bloco `state.headset` tinha `PCM Playback Volume: value 0` para os dois canais salvos.

Ou seja: em algum momento entre julho e hoje (provavelmente no shutdown, quando o
`alsa-state.service` roda `alsactl store` automaticamente), o `alsactl store`
capturou o `PCM` já zerado e sobrescreveu o "100% pinado". A partir daí a regra
udev passou a restaurar zeros a cada conexão — o fix estava se **auto-sabotando**
via arquivo de estado mutável.

Fator agravante: a ordem dos cards mudou de novo (`headset` = card 2 hoje, era
card 0 em julho). Como a regra udev já usava nome (`restore headset`), essa parte
não quebrou — mas confirma que qualquer att pode reordenar.

**Lição:** `/var/lib/alsa/asound.state` é mutável e reescrito automaticamente em
shutdown. Não é confiável como fonte de verdade para "PCM sempre em 100%". O fix
precisa chumbar o valor num lugar imutável — a própria regra udev.

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

Fator agravante descoberto em 2026-07-20: qualquer fix que dependa de
`/var/lib/alsa/asound.state` é frágil, porque `alsa-state.service` roda
`alsactl store` no shutdown. Se por acaso o `PCM,0` estiver zerado nesse
momento (o próprio bug garante que isso vai acontecer), o estado "correto"
é sobrescrito por zeros e o fix passa a **restaurar o bug** a cada conexão.

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

### 2. Aplicar `PCM=100%` agora (fix imediato do estado atual)

```bash
# Sobe os dois canais para 100% (usar o NOME do card, não o índice)
amixer -c headset sset 'PCM' 100% 100%

# Verificar (deve mostrar Front Left e Front Right em 100%)
amixer -c headset get 'PCM'
```

> **Não** rodar `alsactl store` — isso reintroduz a dependência do arquivo mutável
> `asound.state`, que foi justamente a causa da recorrência de 2026-07-20. A
> persistência entre conexões é responsabilidade da regra udev abaixo.

### 3. Regra udev — chumbar `PCM=100%` a cada conexão do dongle

Como o dongle é plugado depois do boot, o `alsa-restore.service` sozinho não basta.
A regra dispara `amixer` **direto** com o valor codificado — não lê nenhum arquivo
de estado, então não pode ser corrompida por `alsactl store` posterior. Match por
`KERNEL=="controlC*"` para disparar só uma vez (quando o control device fica
pronto), e por `ATTRS{idVendor}` / `ATTRS{idProduct}` do dongle (imune a reordenação de card).

**Arquivo:** `/etc/udev/rules.d/90-redragon-headset.rules`

```
ACTION=="add", SUBSYSTEM=="sound", KERNEL=="controlC*", ATTRS{idVendor}=="040b", ATTRS{idProduct}=="0897", RUN+="/usr/bin/amixer -c headset sset PCM 100% 100%"
```

Aplicar:

```bash
sudo udevadm control --reload-rules
```

Testar: desconectar e reconectar o dongle. `amixer -c headset get PCM` deve
mostrar `[100%]` nos dois canais **imediatamente**, sem intervenção manual.

---

## Estado final

- WirePlumber faz todo o volume em software; **nunca** toca no `PCM,0`.
- `PCM,0` fica em 100% FL+FR, forçado pela regra udev a cada conexão do dongle
  (valor chumbado na regra — **não** depende de `asound.state`).
- Regra udev matcha por `idVendor/idProduct` do dongle (imune a reordenação de card)
  e por `KERNEL=="controlC*"` (dispara uma vez, no momento certo).
- Volume funciona em qualquer nível pelo painel do KDE, nos dois canais.
- Sobrevive a reboot, a reconexão do dongle e a `alsactl store` automático no shutdown.

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `~/.config/wireplumber/wireplumber.conf.d/51-h510-soft-mixer.conf` | Criado | Força volume por software no card do dongle (`api.alsa.soft-mixer`) |
| `/etc/udev/rules.d/90-redragon-headset.rules` | Modificado (2026-07-20) | Chama `amixer` direto com `PCM=100%` chumbado na regra — não depende mais de `alsactl restore` nem do `asound.state` |

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

# Corrigir manualmente se necessário (não deveria precisar — a regra udev cuida)
amixer -c headset sset 'PCM' 100% 100%
# NÃO rodar 'alsactl store' — vide seção 2 da Solução.

# Forçar disparo da regra udev sem replugar (útil pra testar)
sudo udevadm trigger --action=add --subsystem-match=sound

# Reaplicar config do WirePlumber
systemctl --user restart wireplumber

# Sinks do PipeWire / card do headset
pactl list sinks short
pactl list cards | grep -A 10 "H510"

# Inspecionar o estado salvo do ALSA (só pra diagnóstico — não é fonte de verdade)
grep -B1 -A10 "^state.headset" /var/lib/alsa/asound.state | head -60
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
- **Não usar `/var/lib/alsa/asound.state` como fonte de verdade** para valores que
  precisam ser garantidos. O `alsa-state.service` roda `alsactl store` no shutdown
  e captura o estado corrente — se o `PCM,0` estiver zerado nesse instante (o
  próprio bug garante que vai estar em algum momento), o "valor correto" é
  sobrescrito por zeros e o fix passa a restaurar o bug. Solução: chumbar o valor
  desejado dentro da própria regra udev (`amixer ... sset PCM 100% 100%`).
- **`RUN+=` em udev deve matchar `KERNEL=="controlC*"`** para eventos de card,
  senão dispara múltiplas vezes (uma para cada subdevice `pcmC*D*p`) e/ou pode
  disparar antes do control device estar pronto para o `amixer`.
```
