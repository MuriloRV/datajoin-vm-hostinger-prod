# 04 — Firewall (ufw + fail2ban)

Firewall de host com `ufw` (deny-default no incoming) + `fail2ban` (ban de IPs por padrão de log). Nenhuma porta exposta na internet pública além de SSH — HTTP/HTTPS sairão via Cloudflare Tunnel (passo 06).

Roda **na VM** (`ssh vm-hostinger`).

## Pré-requisito

SSH key-only configurado (passo 03 ok). Confirmar antes:
```bash
ssh -o BatchMode=yes vm-hostinger "echo ok"   # deve imprimir 'ok'
```

## 1. Instalar pacotes

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y ufw fail2ban
```

`ufw` geralmente já vem na imagem cloud do Ubuntu, mas o comando é idempotente.

## 2. Configurar regras do ufw

**Ordem importa**: definir regras ANTES de habilitar o firewall. Depois `enable` aplica de forma atômica e mantém conexões `ESTABLISHED` (sua sessão SSH atual sobrevive — `before.rules` do ufw libera RELATED/ESTABLISHED por default).

```bash
ufw default deny incoming
ufw default allow outgoing
ufw limit 22/tcp comment 'SSH (rate-limited)'
ufw --force enable
```

- `default deny incoming`: tudo bloqueado por padrão na entrada.
- `default allow outgoing`: outbound livre (pra `apt`, Docker pulls, Cloudflare Tunnel, etc).
- `limit 22/tcp`: rate-limit (~6 conn/30s por IP) usando módulo `recent` do iptables — defesa de baixo custo contra brute-force agressivo.
- `--force`: pula o prompt "this may disrupt SSH".

## 3. Verificar ufw

```bash
ufw status verbose
```

Esperado:
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     LIMIT IN    Anywhere
22/tcp (v6)                LIMIT IN    Anywhere (v6)
```

## 4. Configurar fail2ban

Drop-in em `jail.d/` (não editar `jail.conf` direto — sobrescrito em upgrades):

```bash
cat > /etc/fail2ban/jail.d/datajoin.local <<'EOF'
[DEFAULT]
# Ban duro: 1 hora
bantime = 1h
# Janela de observacao
findtime = 10m
# Numero de falhas antes de banir.
# Mantemos 10 (nao 5) pra reduzir risco de auto-ban por erro de chave.
maxretry = 10
# Backend systemd: le journald (ssh do Ubuntu 22.04 nao escreve mais em /var/log/auth.log por default)
backend = systemd
# Nunca banir loopback
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
port = 22
EOF
```

## 5. Habilitar e iniciar fail2ban

```bash
systemctl enable --now fail2ban
systemctl status fail2ban --no-pager | head -10
fail2ban-client status sshd
```

Esperado: serviço `active (running)`, jail `sshd` ativo, currently banned = 0.

## 6. Verificação ponta-a-ponta

Em outra janela do PowerShell:
```powershell
ssh vm-hostinger "echo 'still in'"
```

Deve continuar funcionando — você ainda passa pelas regras de ufw (limit, não block) e seu IP não tem nenhum failed login no journal.

## Recovery

Se ufw te trancar fora:
- Console VNC da Hostinger → `ufw disable` → investigar.

Se fail2ban banir seu IP:
- `fail2ban-client unban <seu-ip>` (via console ou outra rota).
- Ou desabilitar temporariamente: `systemctl stop fail2ban`.

## Notas pra próximos passos

- `06 — Cloudflare Tunnel` não vai pedir abrir portas TCP externas (o tunnel é outbound).
- `07 — k3s` vai precisar abrir portas no ufw (6443 API, 10250 kubelet, 8472 flannel-vxlan, etc). Idealmente restritas a `127.0.0.1` ou interfaces internas, já que é single-node. Esse doc trata disso.
