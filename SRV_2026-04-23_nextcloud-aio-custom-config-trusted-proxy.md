# Config: Nextcloud AIO — Configurações Persistentes via custom.config.php

- **Data:** 2026-04-23 (atualizado 2026-04-24)
- **Local:** Servidor VPS ({{VPS_HOSTNAME}})
- **Sistema:** Nextcloud AIO (Docker), Nginx Proxy Manager, Tailscale
- **Contexto:** Configs feitas via `occ` são sobrescritas pelo AIO em atualizações. Este runbook documenta configurações persistentes e a arquitetura de rede do setup.

---

## Arquitetura de Rede Completa

Entender o caminho real da requisição é essencial para qualquer diagnóstico:

```
Telefone (Tailscale 100.x.x.x)
  → Host do servidor via Tailscale ({{TAILSCALE_IP_SERVER}})
    → Interface Docker bridge (172.18.0.1) ← gateway da rede Docker
      → Container NPM (172.18.0.5) ← injeta X-Forwarded-For
        → Container nextcloud-aio-apache (172.18.0.7)
          ↓ (internamente, dois servidores no mesmo container)
          → Caddy (porta 11000) ← recebe do NPM, confia em private_ranges
            → Apache interno (127.0.0.1:8000) ← mod_remoteip desabilitado
              → PHP-FPM (container nextcloud-aio-nextcloud, 172.19.0.12)
```

**Ponto crítico:** o container `nextcloud-aio-apache` roda dois servidores internamente — o Caddy (que recebe o tráfego externo) e o Apache (que faz proxy para o PHP-FPM via FastCGI). O Apache não tem `mod_remoteip` habilitado, então o PHP-FPM sempre enxerga `127.0.0.1` como IP de origem, independente do que chega no Caddy.

**Consequência prática:** o bruteforce protection do Nextcloud sempre acumula tentativas em `127.0.0.1` nessa arquitetura — tanto por credenciais erradas em loop quanto pelo comportamento normal do DAVx5 de fazer requisições paralelas ao sincronizar múltiplos calendários. A solução em dois níveis: app passwords para cada cliente de terceiro, e bypass do rate limiting para `127.0.0.1` no painel do Nextcloud.

---

## Variáveis de Ambiente do Mastercontainer (compose.yaml)

```yaml
environment:
  - APACHE_PORT=11000
  - APACHE_IP_BINDING=127.0.0.1      # Apache escuta só no loopback
  - APACHE_ADDITIONAL_NETWORK=proxy  # Apache entra na rede Docker proxy
  - NC_TRUSTED_PROXIES=npm           # mecanismo oficial AIO para trusted proxies
  - NEXTCLOUD_UPLOAD_LIMIT=10G
  - NEXTCLOUD_DATADIR={{NEXTCLOUD_DATADIR}}
```

`NC_TRUSTED_PROXIES=npm` é a variável oficial do AIO para adicionar proxies confiáveis — o AIO resolve o nome do container para o IP correspondente. `APACHE_IP_BINDING=127.0.0.1` restringe o Apache ao loopback interno, e o Caddy faz a interface com o mundo externo.

---

## Por que custom.config.php?

O AIO regenera o `config.php` durante atualizações, sobrescrevendo configs aplicadas via `occ`. O `custom.config.php` é carregado automaticamente junto com o `config.php` principal e nunca é tocado pelo AIO.

**Localização no host:**
```
/var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/custom.config.php
```

---

## Configurações Atuais

### custom.config.php

```php
<?php
$CONFIG = [
  'trusted_proxies' => ['172.18.0.0/16', '172.18.0.1', '127.0.0.1'],
  'forwarded_for_headers' => ['HTTP_X_FORWARDED_FOR'],
];
```

- `172.18.0.0/16` — subnet Docker da rede proxy (cobre NPM e Apache AIO)
- `172.18.0.1` — gateway Docker bridge (IP de entrada do tráfego Tailscale)
- `127.0.0.1` — loopback interno entre Caddy e Apache dentro do container AIO

> `::1`, `172.19.0.0/16` já são gerenciados pelo AIO no `config.php` base.

### NPM: Custom Config do proxy host do Nextcloud

Em **Hosts → Proxy Hosts → {{NEXTCLOUD_DOMAIN}} → Advanced**:

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

## Bypass do Rate Limiting (painel Nextcloud)

Em **Settings → Administration → Security**:

- Toggle **"Bypass rate limiting for allowed IPs"**: **ativado**
- IP na allowlist: `127.0.0.1` com máscara `/32`

**Por que `127.0.0.1` e não o IP real do dispositivo:** a arquitetura AIO sempre apresenta `127.0.0.1` para o Nextcloud, independente de qual dispositivo está fazendo a requisição. Adicionar o IP real do telefone ou notebook não teria efeito. Como o acesso é restrito à rede Tailscale (privada), isentar `127.0.0.1` é equivalente a isentar apenas os dispositivos do usuário.

Esta configuração persiste no banco de dados do Nextcloud e não é afetada por updates do AIO.

--- (DAVx5, FolderSync, etc.)

**Usar sempre App Password**, não a senha principal. App passwords não acionam o bruteforce protection do Nextcloud — essencial dado que clientes como DAVx5 fazem múltiplas requisições paralelas ao sincronizar, e um app com senha errada em loop acumula tentativas e bloqueia `127.0.0.1` para **todos** os outros apps simultaneamente.

Criar em: **Settings → Personal → Security → Devices & Sessions → Create new app password**

Criar um app password separado por app, com nome descritivo:

| App | Função | Nome sugerido |
|---|---|---|
| DAVx5 | CalDAV/CardDAV (contatos, agendas, tasks) | "DAVx5" |
| FolderSync | WebDAV (pastas de notas para Obsidian) | "FolderSync" |
| Desktop client | Sincronização de arquivos | "Desktop" |
| App mobile Nextcloud | App oficial mobile | "Mobile" |

---

## Como Aplicar (setup inicial ou após reinstalação)

```bash
cat > /var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/custom.config.php << 'EOF'
<?php
$CONFIG = [
  'trusted_proxies' => ['172.18.0.0/16', '172.18.0.1', '127.0.0.1'],
  'forwarded_for_headers' => ['HTTP_X_FORWARDED_FOR'],
];
EOF
```

```bash
# Verificar
cat /var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/custom.config.php

# Confirmar que o Nextcloud está lendo (sem duplicatas)
docker exec -it nextcloud-aio-nextcloud php occ config:system:get trusted_proxies
```

Resultado esperado:
```
172.18.0.0/16
172.18.0.1
172.19.0.0/16
127.0.0.1
127.0.0.1   ← duplicata do 127.0.0.1 é inofensiva (AIO já inclui no base)
::1
```

---

## Verificação Pós-Update

```bash
# Confirmar que custom.config.php está sendo lido
docker exec -it nextcloud-aio-nextcloud php occ config:system:get trusted_proxies
# deve incluir 172.18.0.0/16

# Verificar estado do bruteforce (deve estar em 0 após update limpo)
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts 127.0.0.1
```

---

## Como Adicionar Novas Configs

```bash
nano /var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/custom.config.php
```

**Atenção:** usar sempre `>` (sobrescrever) e nunca `>>` (appendar) ao reescrever via shell — um segundo bloco `<?php` no mesmo arquivo causa erro de parse e derruba o Nextcloud.

```php
<?php
$CONFIG = [
  'chave' => 'valor',
  'chave_array' => ['item1', 'item2'],
];
```

Não é necessário reiniciar o container.

---

## Diagnóstico de Bloqueios 429

```bash
# Checar quais IPs estão bloqueados
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts 127.0.0.1
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts 172.18.0.1
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts 172.18.0.5

# Resetar bloqueio
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:reset 127.0.0.1

# Ver Caddyfile em runtime
docker exec -it nextcloud-aio-apache cat /tmp/Caddyfile

# Ver logs de acesso do NPM para o Nextcloud
docker exec -it npm tail -20 /data/logs/proxy-host-3_access.log
```

---

## Arquivos

| Arquivo | Localização no Host | Descrição |
|---|---|---|
| `custom.config.php` | `/var/lib/docker/volumes/nextcloud_aio_nextcloud/_data/config/` | Configs persistentes do Nextcloud |
| `compose.yaml` | `/srv/nextcloud/compose.yaml` | Definição do mastercontainer |

---

## Referência

- Originado do incidente: `SRV_2026-04-23_nextcloud-davx5-429-trusted-proxy-fix.md`
- Documentação oficial AIO reverse proxy: https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
- Documentação Nextcloud config.php: https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/config_sample_php_parameters.html
