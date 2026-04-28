# Fix: Nextcloud — Erro 429 Too Many Requests (DAVx5 / FolderSync)

- **Data:** 2026-04-23 / 2026-04-24
- **Local:** Servidor VPS ({{VPS_HOSTNAME}})
- **Sistema:** Nextcloud AIO (Docker), Nginx Proxy Manager, Tailscale
- **Afetado:** DAVx5 e FolderSync no Android — sincronização de contatos, agendas, tasks e pastas

---

## Problema

DAVx5 e/ou FolderSync reportavam erro `429 Too Many Requests` e não sincronizavam. O Nextcloud funcionava normalmente via browser e via app oficial. O problema era exclusivo à sincronização via DAV.

---

## Causa Raiz Final (descoberta após investigação em dois dias)

Três fatores combinados:

**1. Credenciais desatualizadas em apps de sincronização.** Quando a senha do Nextcloud é trocada, todos os apps que se conectam (DAVx5, FolderSync, app do Nextcloud, desktop client) precisam ser atualizados. Um app com senha errada fica tentando em loop, acumulando falhas rapidamente.

**2. Arquitetura AIO faz tudo aparecer como `127.0.0.1`.** Todo tráfego passa por NPM → Caddy (interno AIO) → Apache (interno AIO) → PHP-FPM. O Apache não propaga o IP real para o PHP, então o Nextcloud registra todas as falhas no mesmo IP (`127.0.0.1`). Ao atingir 10 tentativas, bloqueia `127.0.0.1` — o que afeta **todos** os apps de sincronização simultaneamente.

**3. DAVx5 faz requisições paralelas que sozinhas já atingem o threshold.** Com muitos calendários, o DAVx5 sincroniza múltiplas coleções ao mesmo tempo. Cada uma faz uma requisição inicial sem autenticação, recebe 401, e reenvia com credenciais. Com 6+ calendários em paralelo, as requisições iniciais já acumulam 10+ "falhas" instantaneamente — mesmo com credenciais corretas e app password válido.

**Solução definitiva:** app password para cada app de terceiro + bypass do rate limiting para `127.0.0.1` no painel do Nextcloud.

---

## Linha do Tempo dos Incidentes

### Incidente 1 — 2026-04-23 manhã: primeiro bloqueio

**Causa:** senha do usuário trocada na noite anterior. DAVx5 tentou autenticar com a senha antiga durante a noite, acumulando tentativas de brute force no IP do container NPM (`172.18.0.5`).

**Confirmação:**
```bash
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts 172.18.0.5
# - attempts: 11
# - delay: 25000
```

**Fix aplicado:**
```bash
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:reset 172.18.0.5
# + atualizar senha no DAVx5
# + adicionar NPM como trusted proxy via occ
```

---

### Incidente 2 — 2026-04-23: update do AIO resetou as configs

Update do Nextcloud AIO no mesmo dia sobrescreveu as configs feitas via `occ`. O DAVx5 voltou a receber 429.

**Causa:** configs via `occ config:system:set` não persistem entre updates do AIO.

**Fix aplicado:** migrar para `custom.config.php`.

Ver doc de runbook: `SRV_2026-04-23_nextcloud-aio-custom-config-trusted-proxy.md`

---

### Incidente 3 — 2026-04-23: 429 persistindo, agora em 127.0.0.1

Após migrar para `custom.config.php`, o bloqueio passou a ocorrer em `127.0.0.1` ao invés de `172.18.0.5`.

**Investigação:**

O log do NPM mostrava `[Client 172.18.0.1]` — o gateway Docker bridge, não o IP do telefone. Investigação profunda revelou a arquitetura interna do container `nextcloud-aio-apache`:

```
NPM (172.18.0.5)
  → Caddy (dentro do container nextcloud-aio-apache, porta 11000)
    → Apache interno (127.0.0.1:8000)
      → PHP-FPM (container nextcloud-aio-nextcloud)
```

O Caddy repassa para o Apache via `reverse_proxy 127.0.0.1:8000`. O Apache não tem `mod_remoteip` habilitado e não propaga o `X-Forwarded-For` para o PHP-FPM. O Nextcloud recebe a conexão como `127.0.0.1`.

Adicionado `127.0.0.1` e `172.18.0.1` ao `custom.config.php`. O bloqueio continuava ocorrendo porque o problema não era o `trusted_proxies` — era o comportamento do DAVx5 de fazer múltiplas requisições paralelas.

---

### Incidente 4 — 2026-04-24 manhã: investigação contínua

Retomada da investigação. Descobertas relevantes:

- O compose do mastercontainer tem `NC_TRUSTED_PROXIES=npm` — variável oficial do AIO para trusted proxies
- O `APACHE_IP_BINDING=127.0.0.1` faz o Apache escutar apenas no loopback
- O Caddyfile interno confirma `trusted_proxies static private_ranges 100.64.0.0/10` — Caddy já confia nos ranges privados
- O Redis ficou instável durante o update, causando comportamento errático
- Confirmado via curl que as credenciais do usuário funcionavam normalmente (HTTP 200)
- O bruteforce acumulava 11 tentativas instantaneamente após cada reset

Aplicado App Password no DAVx5 — funcionou por um tempo, mas o bloqueio voltou.

---

### Incidente 6 — 2026-04-24 fim do dia: bloqueio por requisições paralelas do DAVx5

DAVx5 e Nextcloud desktop client do Linux voltaram a receber 429. Desta vez as credenciais estavam corretas em todos os apps.

**Análise do log do NPM:**
```
14:16:54 - DAVx5 recebendo 429 em 6 calendários simultaneamente
14:19:50 - Nextcloud desktop client recebendo 429
14:20:52 - Nextcloud desktop client voltando a funcionar (207)
```

Todas as 6 requisições do DAVx5 chegaram no mesmo segundo com `User-Agent: DAVx5/4.5.10-ose`. O bloqueio expirou sozinho em menos de 2 minutos — as tentativas eram as requisições iniciais sem autenticação do DAVx5 sincronizando 6 calendários em paralelo, não falhas reais de credencial.

**Causa:** comportamento normal do DAVx5 com muitos calendários + threshold de 10 tentativas do bruteforce + arquitetura AIO onde tudo é `127.0.0.1` = bloqueio garantido a cada sync com muitas coleções.

**Fix definitivo:** ativar o toggle **"Bypass rate limiting for allowed IPs"** no painel do Nextcloud em **Settings → Administration → Security**, adicionando `127.0.0.1` com máscara `/32`.

**Por que `127.0.0.1` e não o IP real do dispositivo:** dado que a arquitetura AIO sempre apresenta `127.0.0.1` para o Nextcloud, adicionar o IP real do telefone ou notebook não teria efeito. O bypass precisa ser no IP que o Nextcloud realmente vê. Como o acesso é restrito à rede Tailscale (privada), isentar `127.0.0.1` é equivalente a isentar apenas os dispositivos do usuário.

Resultado: DAVx5, FolderSync e Nextcloud desktop client sincronizando normalmente sem acionar bruteforce.

---

## Estado Final

```
trusted_proxies (config.php base, gerenciado pelo AIO):
  172.18.0.0/16, ::1, 172.19.0.0/16, 127.0.0.1

trusted_proxies (custom.config.php):
  172.18.0.0/16, 172.18.0.1, 127.0.0.1

forwarded_for_headers: HTTP_X_FORWARDED_FOR

NPM proxy host 3.conf Custom Config:
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

Bruteforce bypass (painel Nextcloud → Settings → Administration → Security):
  127.0.0.1/32 na allowlist com toggle "Bypass rate limiting for allowed IPs" ✓

DAVx5: App Password "DAVx5" ✓
FolderSync: App Password "FolderSync" ✓
Nextcloud desktop client (Linux): App Password próprio ✓
```

---

## Como Evitar no Futuro

**Ao trocar a senha do Nextcloud**, atualizar todos os apps que se conectam — DAVx5, FolderSync, Nextcloud desktop client, app mobile. Com app passwords separados por app é mais fácil identificar qual está falhando e revogar individualmente sem afetar os outros.

**Se o 429 ocorrer mesmo com credenciais corretas**, provavelmente é o DAVx5 sincronizando muitos calendários em paralelo — o bloqueio expira sozinho em 1-2 minutos. O bypass do `127.0.0.1` na allowlist do painel previne que isso aconteça.

**Se o 429 persistir por mais de 2-3 minutos**, provavelmente tem um app com credencial errada em loop. Para diagnosticar:

```bash
# Ver os logs do NPM para identificar o culpado pelo User-Agent
docker exec -it npm tail -50 /data/logs/proxy-host-3_access.log | grep 401

# Checar se há bloqueio ativo
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts 127.0.0.1

# Resetar o bloqueio para liberar acesso enquanto investiga
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:reset 127.0.0.1
```

**Apps conectados ao Nextcloud neste setup:**

| App | Tipo | Credencial |
|---|---|---|
| DAVx5 | CalDAV/CardDAV (contatos, agendas, tasks) | App Password "DAVx5" |
| FolderSync | WebDAV (pastas de notas para Obsidian) | App Password "FolderSync" |
| Nextcloud desktop (Linux) | Sincronização de arquivos | App Password próprio |

---

## Arquivos Modificados / Configurações Aplicadas

| Recurso | Tipo | Descrição |
|---|---|---|
| `custom.config.php` | Criado | `trusted_proxies` e `forwarded_for_headers` persistidos entre updates |
| NPM proxy host `3.conf` | Modificado | `X-Forwarded-For $proxy_add_x_forwarded_for` via Custom Config |
| Nextcloud Security panel | Configurado | Bypass rate limiting para `127.0.0.1/32` ativado |

---

## Referência Rápida — Comandos Úteis

```bash
# Checar tentativas de brute force para um IP
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:attempts <IP>

# Resetar bloqueio de brute force para um IP
docker exec -it nextcloud-aio-nextcloud php occ security:bruteforce:reset <IP>

# Ver trusted proxies configurados
docker exec -it nextcloud-aio-nextcloud php occ config:system:get trusted_proxies

# Ver logs do Nextcloud
docker exec -it nextcloud-aio-nextcloud php occ log:tail

# Ver logs de acesso do proxy host do Nextcloud no NPM
docker exec -it npm tail -20 /data/logs/proxy-host-3_access.log

# Ver Caddyfile gerado em runtime pelo AIO
docker exec -it nextcloud-aio-apache cat /tmp/Caddyfile

# Ver config do proxy host do Nextcloud no NPM
docker exec -it npm cat /data/nginx/proxy_host/3.conf

# IP dos containers (para diagnóstico)
docker inspect npm | grep -i ipaddress
docker inspect nextcloud-aio-apache | grep -i ipaddress
```
