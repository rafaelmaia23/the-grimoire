# Guia de Documentação — Grimoire

Este arquivo define as regras e padrões para criar e manter documentação neste repositório. Deve ser lido antes de criar qualquer novo documento — por eu mesmo ou por um agente de IA.

---

## O que é o Grimoire

Coleção pessoal de documentação técnica no estilo runbook/TIL (Today I Learned): registros de problemas resolvidos, configurações aplicadas, decisões tomadas e setups realizados. O foco é ser referência futura para eu mesmo — mas o repositório é público e pode ser usado como portfólio, então tudo que for escrito deve ser seguro para exposição.

---

## Nomenclatura dos Arquivos

```
{TIPO}-{ESCOPO}-{YYYY-MM-DD}-{slug}.md
```

Separador único: hífen (`-`). Sem underscores.

| Parte | Formato | Exemplo |
|---|---|---|
| `TIPO` | Prefixo em maiúsculas, ver tabela abaixo | `INC`, `SETUP`, `ADR` |
| `ESCOPO` | Prefixo em maiúsculas, onde foi aplicado | `PC`, `SRV`, `NOTE`, `OCI` |
| `YYYY-MM-DD` | Data | `2026-05-22` |
| `slug` | Kebab-case em português, descritivo | `kvm-libvirt-coexistencia-docker` |

**Exemplo completo:**
```
SETUP-PC-2026-05-22-qemu-kvm-libvirt-com-docker.md
INC-PC-2026-05-21-libvirt-virbr0-docker-forward-drop.md
ADR-PC-2026-05-22-libvirt-rede-isolada-do-docker.md
```

### Tipos disponíveis

| Tipo | Propósito | Pergunta-teste |
|---|---|---|
| `SETUP` | Instalação/configuração de algo do zero. Como reinstalar/refazer. | "Se eu formatar a máquina, preciso disso pra montar de novo?" |
| `FIX` | Resolução de problema pontual e enxuto. | "Algo quebrou, eu consertei, como?" |
| `INC` | Incidente em profundidade — timeline, investigação, causa raiz, lições. | "Foi grave/duradouro/complexo, vale guardar a história completa?" |
| `ADR` | Decisão arquitetural. Por que escolhi X e não Y. | "Por que esse caminho e não outro?" |
| `RUNBOOK` | Procedimento operacional recorrente. | "Vou executar isso de novo, várias vezes?" |
| `REF` | Referência/cheat sheet. Comandos, atalhos, decisões fixas. | "É consulta rápida, não passo-a-passo?" |

**Distinções importantes:**

- **FIX vs INC:** FIX é enxuto (1-2 sintomas, causa direta, comando que resolveu). INC tem timeline, investigação, múltiplos sintomas, causa raiz aprofundada. Quando em dúvida, comece como FIX — se virar épico durante a escrita, promove pra INC.
- **SETUP vs RUNBOOK:** SETUP é instalação inicial, geralmente executada 1x (ou em reinstalações). RUNBOOK é procedimento recorrente (backup mensal, troca de servidor Mullvad, etc.).
- **REF vs todo o resto:** REF é consultivo, sem narrativa. Listas de comandos, atalhos do KDE, snippets. Se você abre o doc pra "olhar uma coisa rápida", é REF.

### Escopos disponíveis

| Escopo | Uso |
|---|---|
| `PC` | Computador pessoal |
| `SRV` | Servidor VPS (Hostinger) |
| `NOTE` | Notebook pessoal |
| `OCI` | VM no Oracle Cloud Infrastructure |

Para adicionar novo escopo, documente aqui antes de usar.

---

## Estrutura do Documento

### Cabeçalho obrigatório

```markdown
# {TIPO}: {Descrição curta}

- **Data:** YYYY-MM-DD
- **Local:** {onde foi aplicado}
- **Sistema:** {OS, versão, ambiente relevante}
```

O `{TIPO}` no `# H1` é o mesmo do prefixo do arquivo, em capitalização normal: `SETUP`, `FIX`, `INC`, `ADR`, `RUNBOOK`, `REF`.

### Seções por tipo

Cada tipo de doc tem seções esperadas. Não é rigido — use o que faz sentido.

**SETUP**
- Contexto — por que esse setup
- Pré-requisitos — o que precisa estar pronto antes
- Verificação prévia — confirmar estado atual antes de aplicar
- Instalação / Como configurar — passos
- Verificação pós-setup — confirmar que ficou ok
- Estado final
- Arquivos criados/modificados

**FIX**
- Sintoma — o que estava errado
- Causa raiz — por quê
- Solução — o que foi feito
- Verificação
- Arquivos modificados (se aplicável)

**INC**
- Contexto / Sintoma inicial
- Linha do tempo (se aplicável) — investigação cronológica
- Causa raiz
- Solução final
- Estado final
- Como evitar no futuro
- Arquivos modificados

**ADR**
- Contexto — situação que motivou a decisão
- Decisão — o que foi escolhido
- Alternativas consideradas — o que foi descartado e por quê
- Consequências — trade-offs, o que muda no futuro
- Status — proposta / aceita / superada / revogada

**RUNBOOK**
- Quando executar — gatilhos
- Pré-requisitos
- Procedimento — passo a passo
- Verificação
- Rollback (se aplicável)

**REF**
- Tópico — agrupamento dos comandos/atalhos
- Comandos com explicação curta

### Seção "Arquivos criados/modificados" (obrigatória quando aplicável)

Quando o documento criou ou modificou arquivos no sistema, sempre terminar com:

```markdown
## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/caminho/do/arquivo` | Criado / Modificado / Configurado | O que faz |
```

REF e ADR podem dispensar essa seção se forem puramente consultivos.

---

## Segurança — Regra dos Placeholders

**Nunca commitar valores reais de:**
- Endereços IP de servidores
- Nomes de usuário em servidores remotos
- Domínios próprios
- Caminhos de armazenamento que revelem estrutura interna
- Tokens, senhas, chaves de API

**Substituir por placeholders no formato `{{NOME_EM_MAIUSCULAS}}`:**

```markdown
# Em vez disso:
ssh rmf@72.60.5.59

# Escrever assim:
ssh {{VPS_USER}}@{{VPS_IP}}
```

**Registrar o valor real em `values.local`** (arquivo gitignored):

```
VPS_USER=nome-user
VPS_IP=76.61.2.79
```

### O que NÃO precisa de placeholder

- IPs de redes Docker internas padrão (172.17.x, 172.18.x) — são genéricos
- Ranges de rede padrão (192.168.0.0/16, 10.0.0.0/8, 100.64.0.0/10)
- Rede default do libvirt (192.168.122.0/24)
- Servidores DNS públicos (1.1.1.1, 8.8.8.8)
- Versões de software
- Nomes de containers Docker definidos pelo próprio software (ex: `nextcloud-aio-nextcloud`)

### Adicionando um novo placeholder

1. Escolher um nome descritivo em maiúsculas: `{{NOME_DO_VALOR}}`
2. Usar o placeholder no documento
3. Adicionar a entrada correspondente em `values.local`

---

## Estilo de Escrita

- **Idioma:** Português brasileiro
- **Tom:** Técnico e direto — como você explicaria para si mesmo no futuro
- **Tempo verbal:** Passado para relatos de incidente, presente/imperativo para instruções
- **Comentários em código:** Mínimos e só quando o porquê não é óbvio
- **Emojis:** Não usar em documentos técnicos
- **Commits:** Um doc por commit, mensagem descritiva

### Estrutura de blocos de código

Sempre especificar a linguagem:

````markdown
```bash
comando aqui
```

```php
<?php código aqui
```

```json
{ "chave": "valor" }
```
````

---

## Checklist antes de commitar

- [ ] Nome do arquivo segue o padrão `{TIPO}-{ESCOPO}-{YYYY-MM-DD}-{slug}.md`
- [ ] Tipo do arquivo coincide com o `# H1` interno
- [ ] Cabeçalho com Data, Local e Sistema preenchidos
- [ ] Nenhum IP real, domínio próprio, usuário ou path sensível no texto
- [ ] Valores sensíveis substituídos por `{{PLACEHOLDER}}` e registrados em `values.local`
- [ ] Tabela "Arquivos criados/modificados" no final (quando aplicável)
- [ ] `values.local` está no `.gitignore` e **não** está sendo commitado

---

## Backup do `values.local`

O `values.local` não é commitado — fazer backup manual periodicamente:
- Copiar para o cofre de senhas (ex: Bitwarden como Secure Note)
- E/ou manter cópia em pendrive criptografado

Localização: raiz do repositório do grimoire.
