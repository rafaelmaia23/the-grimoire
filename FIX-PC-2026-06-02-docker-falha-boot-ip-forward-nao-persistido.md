# FIX: Docker falha no boot — net.ipv4.ip_forward não persistido

- **Data:** 2026-06-02
- **Local:** PC pessoal
- **Sistema:** Fedora 44, Docker 29.5.2

---

## Sintoma

Após reboot, o Docker não sobe automaticamente. `systemctl status docker` mostra `failed (Result: exit-code)` com `restart counter is at 3` — ou seja, tentou 3 vezes e desistiu.

---

## Causa raiz

O `net.ipv4.ip_forward` não estava configurado para persistir entre reboots. O Docker tentava subir antes que qualquer outra coisa aplicasse o valor, e falhava ao tentar criar a rede bridge padrão:

```
failed to start daemon: Error initializing network controller: error creating default "bridge" network:
IPv4 forwarding is disabled: check your host's firewalling and set sysctl net.ipv4.ip_forward=1
```

O serviço estava corretamente habilitado (`systemctl is-enabled docker` → `enabled`), e o problema **não era** race condition — o sysctl simplesmente nunca havia sido persistido. Provavelmente havia sido ativado manualmente em alguma sessão anterior via `sysctl -w net.ipv4.ip_forward=1`, valor que não sobrevive ao reboot.

Confirmação:

```bash
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 0

grep -r "ip_forward" /etc/sysctl.conf /etc/sysctl.d/
# (vazio — nenhuma config persistindo o valor)
```

---

## Solução

Persistir o valor via `/etc/sysctl.d/`:

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl --system
```

Verificar que aplicou:

```bash
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1
```

Subir o Docker:

```bash
sudo systemctl start docker
systemctl status docker
```

---

## Verificação

```bash
# ip_forward ativo
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1

# Docker rodando
systemctl is-active docker
# active
```

Após o próximo reboot, o arquivo em `/etc/sysctl.d/` é lido pelo systemd durante o boot antes dos serviços de rede, garantindo que o Docker encontre o ip_forward ativo ao inicializar.

---

## Arquivos criados/modificados

| Arquivo | Tipo | Descrição |
|---|---|---|
| `/etc/sysctl.d/99-ip-forward.conf` | Criado | Persiste `net.ipv4.ip_forward = 1` entre reboots |
