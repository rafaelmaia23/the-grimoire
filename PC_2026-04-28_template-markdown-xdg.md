# Setup: Template XDG — Criar Arquivo Markdown via Menu de Contexto

- **Data:** 2026-04-28
- **Local:** PC pessoal
- **Sistema:** Fedora 43, KDE Plasma 6.x, Dolphin

---

## Contexto

O KDE Dolphin (e outros gerenciadores de arquivos) suporta o diretório XDG `~/Templates/` para exibir opções de "Criar novo arquivo" no menu de contexto. Adicionado um template para criar arquivos `.md` em branco diretamente pelo botão direito.

---

## Estrutura criada

```
~/Templates/
├── md.desktop          ← entrada do menu de contexto
└── .source/
    └── blank.md        ← arquivo em branco copiado como template
```

---

## Arquivos

**`~/Templates/md.desktop`**

```ini
[Desktop Entry]
Name=Markdown File...
Comment=New Markdown file:
Type=Link
URL=.source/blank.md
Icon=text-markdown
```

**`~/Templates/.source/blank.md`** — arquivo vazio (conteúdo inicial em branco).

---

## Como usar

No Dolphin: clique direito em qualquer pasta → **Criar novo** → **Markdown File...**

O arquivo criado é uma cópia do `blank.md` (em branco), pronto para edição.

---

## Recriar do zero

```bash
mkdir -p ~/Templates/.source
touch ~/Templates/.source/blank.md

cat > ~/Templates/md.desktop << 'EOF'
[Desktop Entry]
Name=Markdown File...
Comment=New Markdown file:
Type=Link
URL=.source/blank.md
Icon=text-markdown
EOF
```

Sem necessidade de reiniciar — o Dolphin reconhece o template imediatamente.

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `~/Templates/md.desktop` | Criado | Entrada do menu de contexto do Dolphin |
| `~/Templates/.source/blank.md` | Criado | Arquivo em branco usado como base do template |
