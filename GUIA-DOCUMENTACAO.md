# Guia de Documentação — Grimoire

Este arquivo define as regras e padrões para criar e manter documentação neste repositório. Deve ser lido antes de criar qualquer novo documento — por eu mesmo ou por um agente de IA.

---

## O que é o Grimoire

Coleção pessoal de documentação técnica no estilo runbook/TIL (Today I Learned): registros de problemas resolvidos, configurações aplicadas e setups realizados. O foco é ser referência futura para eu mesmo — mas o repositório é público e pode ser usado como portifólio, então tudo que for escrito deve ser seguro para exposição.

---

## Nomenclatura dos Arquivos

```
{ESCOPO}_{DATA}_{slug}.md
```

| Parte | Formato | Exemplo |
|---|---|---|
| `ESCOPO` | Prefixo em maiúsculas | `PC`, `SRV`, `NOTE` |
| `DATA` | `YYYY-MM-DD` | `2026-04-23` |
| `slug` | Kebab-case em português, descritivo | `rustdesk-flatpak-to-rpm-fedora43` |

**Escopos disponíveis:**

| Escopo | Uso |
|---|---|
| `PC` | Configurações e fixes no computador pessoal |
| `SRV` | Configurações e fixes no servidor VPS |
| `NOTE` | Configurações e fixes no notebook pessoal |

Para adicionar um novo escopo, documente aqui antes de usar.

---

## Estrutura do Documento

### Cabeçalho obrigatório

```markdown
# {Tipo}: {Descrição curta}

- **Data:** YYYY-MM-DD
- **Local:** {onde foi aplicado}
- **Sistema:** {OS, versão, ambiente relevante}
```

**Tipos:**

| Tipo | Quando usar |
|---|---|
| `Fix:` | Resolução de um problema específico |
| `Setup:` | Configuração de algo do zero |
| `Config:` | Documentação de uma decisão de configuração |

### Seções comuns (usar conforme o tipo de doc)

- **Contexto / Problema** — o que motivou o documento
- **Causa Raiz** — por que o problema ocorria (para docs de Fix)
- **Solução / Instalação / Como configurar** — os passos executados
- **Como usar** — instruções de uso cotidiano (quando aplicável)
- **Estado final** — como ficou depois de tudo aplicado
- **Verificação / Referência rápida** — comandos úteis para checar o estado
- **Como Evitar no Futuro** — lições aprendidas (para docs de Fix)
- **Arquivos criados/modificados** — tabela resumo no final

### Seção "Arquivos criados/modificados" (obrigatória)

Sempre terminar o documento com esta tabela:

```markdown
## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/caminho/do/arquivo` | Criado / Modificado / Configurado | O que faz |
```

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

- [ ] Nome do arquivo segue o padrão `{ESCOPO}_{DATA}_{slug}.md`
- [ ] Cabeçalho com Data, Local e Sistema preenchidos
- [ ] Nenhum IP real, domínio próprio, usuário ou path sensível no texto
- [ ] Valores sensíveis substituídos por `{{PLACEHOLDER}}` e registrados em `values.local`
- [ ] Tabela "Arquivos criados/modificados" no final
- [ ] `values.local` está no `.gitignore` e **não** está sendo commitado

---

## Backup do `values.local`

O `values.local` não é commitado — fazer backup manual periodicamente:
- Copiar para o cofre de senhas (ex: Bitwarden como Secure Note)
- E/ou manter cópia em pendrive criptografado

Localização: raiz do repositório do grimoire.
