# Config: Bash — HISTCONTROL=ignorespace

- **Data:** 2026-04-28
- **Local:** PC pessoal
- **Sistema:** Fedora 43

---

## Contexto

Adicionado `HISTCONTROL=ignorespace` ao `~/.bashrc` para impedir que comandos precedidos por espaço sejam gravados no histórico do Bash. Útil ao rodar comandos com tokens, senhas ou dados sensíveis diretamente no terminal.

---

## Como usar

Prefixar o comando com um espaço antes de digitar:

```bash
 export API_TOKEN=meu_token_secreto
 curl -H "Authorization: Bearer meu_token" https://api.exemplo.com
```

Comandos assim não aparecem em `~/.bash_history` nem em `history`.

---

## Configuração

Linha adicionada ao `~/.bashrc`:

```bash
export HISTCONTROL=ignorespace
```

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `~/.bashrc` | Modificado | Adicionado `export HISTCONTROL=ignorespace` |
