# Setup: SSH — Restauração de Chaves do Notebook Antigo

- **Data:** 2026-04-28
- **Local:** PC pessoal
- **Sistema:** Fedora 43, KDE Plasma 6.x, Wayland

---

## Contexto

Durante a configuração do PC novo, as chaves SSH existentes do notebook antigo foram restauradas a partir de um backup em `/home/rfonseca/Nextcloud copy/backup-configs-pc/.ssh/`. O notebook antigo não foi perdido nem comprometido, então não houve necessidade de gerar novas chaves ou revogar as existentes.

As chaves já estavam registradas no GitHub e no VPS da Hostinger, então funcionam imediatamente após a restauração.

---

## Chaves restauradas

| Arquivo | Tipo | Uso |
|---|---|---|
| `github-fedora` | Chave privada Ed25519 | Autenticação SSH no GitHub |
| `github-fedora.pub` | Chave pública | Registrada nas GitHub SSH Keys da conta |
| `hostinger-maiahub-vps` | Chave privada Ed25519 | Acesso SSH ao VPS da Hostinger |
| `hostinger-maiahub-vps.pub` | Chave pública | Registrada em `~/.ssh/authorized_keys` no VPS |
| `config` | Configuração SSH | Mapeia hosts para as chaves corretas |

---

## Conteúdo do `~/.ssh/config`

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github-fedora
  IdentitiesOnly yes

Host maiahub-vps
  HostName {{VPS_IP}}
  User {{VPS_USER}}
  IdentityFile ~/.ssh/hostinger-maiahub-vps
```

Uso: `ssh maiahub-vps` conecta ao VPS sem precisar especificar usuário ou chave.

---

## Como restaurar (passos)

```bash
# Copiar chaves do backup
cp /caminho/do/backup/.ssh/github-fedora ~/.ssh/
cp /caminho/do/backup/.ssh/github-fedora.pub ~/.ssh/
cp /caminho/do/backup/.ssh/hostinger-maiahub-vps ~/.ssh/
cp /caminho/do/backup/.ssh/hostinger-maiahub-vps.pub ~/.ssh/
cp /caminho/do/backup/.ssh/config ~/.ssh/

# Permissões corretas (obrigatório — SSH recusa chaves com permissões abertas)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/github-fedora ~/.ssh/hostinger-maiahub-vps ~/.ssh/config
chmod 644 ~/.ssh/github-fedora.pub ~/.ssh/hostinger-maiahub-vps.pub
```

---

## Verificação

```bash
# Testar GitHub
ssh -T git@github.com
# Esperado: "Hi <usuário>! You've successfully authenticated..."

# Testar VPS
ssh maiahub-vps
```

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `~/.ssh/github-fedora` | Criado | Chave privada para GitHub |
| `~/.ssh/github-fedora.pub` | Criado | Chave pública para GitHub |
| `~/.ssh/hostinger-maiahub-vps` | Criado | Chave privada para VPS Hostinger |
| `~/.ssh/hostinger-maiahub-vps.pub` | Criado | Chave pública para VPS Hostinger |
| `~/.ssh/config` | Criado | Configuração de hosts SSH |
