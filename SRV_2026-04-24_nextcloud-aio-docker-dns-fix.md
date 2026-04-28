# Fix: Nextcloud AIO — cURL error 6 (DNS não resolve hosts externos nos containers)

- **Data:** 2026-04-24
- **Local:** Servidor VPS ({{VPS_HOSTNAME}})
- **Sistema:** Nextcloud AIO (Docker), AdGuard Home, Nginx Proxy Manager, Tailscale
- **Afetado:** Todos os containers da stack `nextcloud-aio` — erros `cURL error 6` no `nextcloud.log`

---

## Problema

O `nextcloud.log` registrava erros recorrentes de resolução DNS para hosts externos:

```
cURL error 6: Could not resolve host: huggingface.co (DNS server returned general failure)
(see https://curl.haxx.se/libcurl/c/libcurl-errors.html) for https://huggingface.co/blog
```

O Nextcloud funcionava normalmente para o usuário, mas features que dependem de acesso externo (app store, AI assistant, feed de novidades, etc.) falhavam silenciosamente.

---

## Diagnóstico

### Passo 1 — Confirmar que o problema era dentro do container

```bash
docker exec -it nextcloud-aio-nextcloud bash
nslookup huggingface.co
```

Resultado:
```
;; Got SERVFAIL reply from 127.0.0.11
Server:         127.0.0.11
Address:        127.0.0.11#53
** server can't find huggingface.co: SERVFAIL
```

### Passo 2 — Confirmar que o host resolvia normalmente

```bash
nslookup huggingface.co 1.1.1.1
# → resolveu corretamente com múltiplos IPs
```

Conclusão: problema isolado nos containers Docker, não no servidor.

### Passo 3 — Inspecionar configuração DNS da rede e dos containers

```bash
docker network inspect nextcloud-aio
# → campo "Options": {} — sem DNS customizado na rede

docker exec nextcloud-aio-nextcloud cat /etc/resolv.conf
# → nameserver 127.0.0.11
# → # NO EXTERNAL NAMESERVERS DEFINED

cat /etc/docker/daemon.json
# → apenas log-driver e log-opts — sem "dns"

cat /etc/resolv.conf
# → cat: /etc/resolv.conf: No such file or directory
```

---

## Causa Raiz

O Docker estava usando exclusivamente o resolver interno (`127.0.0.11`) sem nenhum upstream DNS configurado. O host não tinha `/etc/resolv.conf` (provavelmente usa `systemd-resolved` ou equivalente), então o Docker não tinha como herdar servidores DNS do sistema.

O resultado: o `127.0.0.11` retornava `SERVFAIL` para qualquer domínio externo porque não sabia para onde encaminhar as queries.

**O AdGuard Home não era o culpado** — apesar de estar rodando como DNS na porta 53 do host, ele não estava sendo usado pelos containers. O tráfego DNS nunca chegava até ele.

---

## Fix

Adicionar DNS upstream no `daemon.json` do Docker:

```bash
nano /etc/docker/daemon.json
```

Conteúdo final do arquivo:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["1.1.1.1", "8.8.8.8"]
}
```

```bash
systemctl restart docker
```

### Verificação

```bash
docker exec nextcloud-aio-nextcloud nslookup huggingface.co
# → resolveu corretamente
```

---

## Observações

- O fix se aplica a **todos** os containers Docker no servidor, não só ao Nextcloud AIO.
- O DNS configurado no `daemon.json` é usado como upstream pelo resolver interno `127.0.0.11` — o mecanismo de resolução de nomes de containers por hostname (ex: `nextcloud-aio-nextcloud`) continua funcionando normalmente.
- Se futuramente quiser que os containers passem pelo AdGuard, a alternativa seria usar o IP do container do AdGuard na rede Docker como DNS, ou o IP da interface bridge do host. Por ora, usar `1.1.1.1` direto é mais simples e resiliente.
- O `mastercontainer` (`nextcloud/all-in-one:latest`) estava em estado `Restarting` durante o diagnóstico — os outros containers do AIO continuaram saudáveis.

---

## Arquivos Modificados

| Arquivo | Localização no Host | Alteração |
|---|---|---|
| `daemon.json` | `/etc/docker/daemon.json` | Adicionado `"dns": ["1.1.1.1", "8.8.8.8"]` |

---

## Referência Rápida — Diagnóstico DNS em Containers

```bash
# Testar resolução DNS dentro de um container
docker exec -it <container> nslookup <domínio>

# Ver DNS configurado na rede Docker
docker network inspect <rede>

# Ver resolv.conf gerado pelo Docker para o container
docker exec <container> cat /etc/resolv.conf

# Ver configuração global do daemon Docker
cat /etc/docker/daemon.json

# Ver DNS do host
cat /etc/resolv.conf
```
