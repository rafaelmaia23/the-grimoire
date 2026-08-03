# Fix: Redragon Zeus Pro (H510-PRO) — Áudio via dongle mudo abaixo de 100% no Linux

- **Data (revisão):** 2026-08-03
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

### Terceira recorrência (2026-07-26, após nova att do sistema)
Áudio via dongle mudo de novo. Diagnóstico — **causa nova, não relacionada às anteriores**:

- `soft-mixer = true` continuava aplicado corretamente ao card (confirmado via `pw-dump`).
- `amixer -c headset get PCM` mostrava `[0%]` nos dois canais — PCM zerado, mudo total.
- **Desta vez o `asound.state` NÃO era o culpado** — o bloco `state.headset` nem tinha entrada de PCM salva.
- A regra udev estava **quebrada por uma quebra de linha espúria no meio do `RUN+=`**,
  logo após `sset`. `wc -l` mostrava 2 linhas onde deveria haver 1, e o
  `journalctl -b` do boot registrava o systemd-udevd rejeitando **as duas**:
  ```
  /etc/udev/rules.d/90-redragon-headset.rules:1 Invalid key/value pair, ignoring.
  /etc/udev/rules.d/90-redragon-headset.rules:2 Invalid key/value pair, ignoring.
  ```
  Ou seja: a regra nunca disparava na conexão do dongle → o `amixer 100%` nunca rodava
  → PCM ficava em 0% → mudo.

Provável origem da quebra: artefato de edição/cópia (hard-wrap em ~80 colunas caindo
exatamente depois de `sset`). Não foi regressão do sistema nem reordenação de card
(o card continuou `headset`); foi o arquivo da regra virar lixo sintático.

**Correção:** reescrever a regra numa **única linha** e recarregar o udev. Verificado
via `cat -A` (nenhum `$` no meio da linha), `journalctl` sem `Invalid key/value pair`,
e teste ao vivo zerando o PCM + `udevadm trigger` restaurando os 100%.

**Lição:** a regra udev tem que ser **uma linha só**. Sempre validar após editar com
`wc -l` (deve ser 1) e `cat -A` (o `$` de fim de linha só pode aparecer no final).
Um `journalctl -b | grep 90-redragon` que mostre `Invalid key/value pair` é o sinal
inequívoco de que a regra está quebrada e não dispara.

### Quarta recorrência (2026-08-03, após att do systemd para 259.8) — causa nova: `%` sem escape em `RUN+=`

Sintoma reportado: fone sem áudio de novo. Hipótese aventada antes de investigar: o
serviço/regra tenta achar o dongle no boot e, como o dongle costuma estar desconectado
nesse momento (o usuário só conecta quando vai usar), talvez isso quebre a regra.
**Essa hipótese não se confirmou** — a causa real foi outra, e o timing do plug do
dongle é irrelevante para ela (ver diagnóstico abaixo).

Diagnóstico:

- Arquivo da regra: íntegra, uma linha só, sem quebra espúria (`wc -l` = 1, `cat -A`
  sem `$` no meio). Ou seja, não é repetição da recorrência de 07-26.
- `journalctl -b 0` mostrou um erro **diferente** de todas as recorrências anteriores,
  disparado no **parse do arquivo pelo `systemd-udevd`, logo no início do boot**
  (11:36:17 e 11:36:39) — **antes** do dongle sequer ser enumerado (11:36:53):
  ```
  /etc/udev/rules.d/90-redragon-headset.rules:1 Invalid value
  "/usr/bin/amixer -c headset sset PCM 100% 100%" for RUN
  (char 40: invalid substitution type), ignoring.
  ```
  O caractere 40 é o primeiro `%` de `100%`. Para o `udev`, `%` dentro de `RUN+=` é
  prefixo de especificador de substituição (`%k`, `%p`, `%n`...); um `%` "solto"
  (seguido de espaço, não de um especificador válido) sempre foi tecnicamente inválido,
  mas versões antigas do `systemd-udevd` toleravam isso. `rpm -qa --last` confirmou que
  o pacote `systemd` (e `systemd-udev` junto) foi atualizado **nesse mesmo boot** para
  `259.8-1.fc44` — essa versão passou a tratar isso como **erro fatal de parse**, e
  descarta a regra inteira (`ignoring`), não só o `%` ambíguo.
- Confirmado também: como a falha é no parse do arquivo (acontece uma vez, quando o
  `systemd-udevd` carrega as regras), ela independe de o dongle estar plugado no boot
  ou ser plugado depois — **a hipótese do usuário sobre timing de conexão não se
  aplica a esta causa**. Pode ainda ser uma preocupação legítima para robustez geral,
  mas não foi o que quebrou desta vez.
- Por que "voltou sozinho" antes da investigação: o `PCM,0` foi encontrado em 100%
  mesmo com a regra confirmadamente quebrada (não disparou em nenhuma das duas
  conexões do dongle detectadas no boot) e sem nenhuma entrada de `PCM` salva no
  `asound.state` para restaurar. Ou seja, não foi o fix rodando — foi o hardware
  calhando de subir no estado alto (~100) no power-on-reset dessa vez, em vez do
  estado mudo (0). Sorte, não correção — o fix continuava morto e o próximo
  replug/boot podia voltar a mutar a qualquer momento.

**Correção:** escapar o `%` como `%%` no valor de `RUN+=` (a forma que o `udev` exige
para um `%` literal):

```
ACTION=="add", SUBSYSTEM=="sound", KERNEL=="controlC*", ATTRS{idVendor}=="040b", ATTRS{idProduct}=="0897", RUN+="/usr/bin/amixer -c headset sset PCM 100%% 100%%"
```

Verificado: `cat -A` (uma linha, sem `$` no meio), PCM zerado manualmente para simular
o bug, `sudo udevadm trigger --action=add --subsystem-match=sound` disparando a regra
e restaurando `[100%]` nos dois canais, e `journalctl` sem nenhum `Invalid value`/
`Invalid key/value pair` após o reload.

**Lição:** `%` dentro de `RUN+=` do udev **precisa ser escapado como `%%`** — não é
apenas estilo, passou a ser **obrigatório** a partir do systemd 259.8 (antes era
tolerado). Qualquer regra udev com `RUN+=` que tenha `%` literal (percentuais,
por exemplo) deve usar `%%`. Após qualquer atualização do `systemd`/`systemd-udev`,
vale reconferir `journalctl -b | grep 90-redragon` por `Invalid value` (parse
rejeitado) além de `Invalid key/value pair` (quebra de linha) — são dois modos de
falha distintos com a mesma mensagem final ("ignoring").

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
ACTION=="add", SUBSYSTEM=="sound", KERNEL=="controlC*", ATTRS{idVendor}=="040b", ATTRS{idProduct}=="0897", RUN+="/usr/bin/amixer -c headset sset PCM 100%% 100%%"
```

> O `%` tem que ser escapado como `%%` — para o `udev`, `%` dentro de `RUN+=` é prefixo
> de especificador de substituição. A partir do systemd 259.8 (recorrência de
> 2026-08-03), um `%` literal sem escape faz o `udev` rejeitar a regra inteira no parse
> (`Invalid value ... invalid substitution type, ignoring`).

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
| `/etc/udev/rules.d/90-redragon-headset.rules` | Modificado (2026-07-20; recriado 2026-07-26; `%%` escapado em 2026-08-03) | Chama `amixer` direto com `PCM=100%` chumbado na regra — não depende mais de `alsactl restore` nem do `asound.state`. **Deve ser 1 linha só** (quebra de linha invalida a regra) e o `%` **deve** ser `%%` (systemd ≥ 259.8 rejeita `%` sem escape). |

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

# A regra udev está íntegra?
wc -l /etc/udev/rules.d/90-redragon-headset.rules      # DEVE ser 1 (recorrência 07-26: quebra de linha)
cat -A /etc/udev/rules.d/90-redragon-headset.rules     # o '$' só pode aparecer no FIM da linha; e o RUN+= deve ter '%%' (não '%' solto)
journalctl -b | grep 90-redragon                       # 'Invalid key/value pair' (quebra de linha) OU
                                                        # 'Invalid value ... invalid substitution type' (recorrência 08-03: '%' sem escape) = regra quebrada, não dispara

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
- **A regra udev tem que ser UMA LINHA só** (regressão de 2026-07-26). Uma quebra de
  linha no meio do `RUN+=` faz o udev rejeitar as duas metades silenciosamente
  (`Invalid key/value pair, ignoring` no `journalctl -b`) e a regra nunca dispara —
  áudio mudo, sem erro óbvio. Após qualquer edição do arquivo, validar com
  `wc -l` (= 1) e `cat -A` (sem `$` no meio). Ao reescrever pela CLI, usar
  `printf '%s\n' '<regra>'` (não `echo` com aspas que possam ser quebradas por hard-wrap).
- **`%` dentro de `RUN+=` tem que ser `%%`** (regressão de 2026-08-03, systemd 259.8).
  `udev` trata `%` como prefixo de especificador (`%k`, `%p`...); um `%` solto sempre
  foi tecnicamente inválido, mas só passou a ser **erro fatal de parse** (regra inteira
  descartada) a partir dessa versão. Atualizações do `systemd`/`systemd-udev` são,
  portanto, um segundo gatilho de recorrência além de att do Plasma — sempre reconferir
  a regra após qualquer att que toque `systemd`.
```
