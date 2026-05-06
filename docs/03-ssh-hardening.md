# 03 — SSH hardening

Trava o SSH em **autenticação só por chave** — desabilita login por senha. Mantém root como usuário válido (a decisão de criar usuário não-root foi adiada — ver README).

Roda **na VM** (`ssh vm-hostinger`).

## Pré-requisito

Antes de aplicar, **a chave SSH precisa estar funcionando**: `ssh vm-hostinger "echo ok"` retorna `ok` sem pedir senha. Se não estiver, voltar pro [`01-ssh-key.md`](01-ssh-key.md). Senão você se tranca pra fora.

## Por que `10-datajoin-hardening.conf` (prefixo baixo)

`/etc/ssh/sshd_config` faz `Include /etc/ssh/sshd_config.d/*.conf` e os drop-ins são lidos em ordem alfabética. Vence a **primeira ocorrência** de cada diretiva. A imagem da Hostinger vem com:

- `50-cloud-init.conf` → `PasswordAuthentication yes`
- `60-cloudimg-settings.conf` → `PasswordAuthentication no`

Por isso o atual estado é "senha permitida" (50 < 60). Pra cravar `no` definitivamente, o nosso drop-in precisa vir **antes** dos dois — daí `10-`.

## 1. Criar o drop-in

```bash
cat > /etc/ssh/sshd_config.d/10-datajoin-hardening.conf <<'EOF'
# datajoin hardening: chave-only, root permitido apenas via chave.
# Wins por precedencia (10- < 50-cloud-init.conf).

# Sem auth por senha em hipotese alguma
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no

# Pubkey auth explicito (default e yes, mas seja explicito)
PubkeyAuthentication yes

# Root pode entrar, mas SO via chave (nunca via senha)
PermitRootLogin prohibit-password

# Janelas curtas pra reduzir brute-force
MaxAuthTries 3
LoginGraceTime 30

# Diversos hardenings que nao quebram nada em VPS standard
X11Forwarding no
PrintMotd no
EOF
```

## 2. Validar sintaxe ANTES de reload

```bash
sshd -t
```

Saída esperada: vazia (silencioso = válido). Qualquer erro **é blocker**: NÃO reloadar com config inválida — sshd pode falhar ao recarregar e te trancar fora. Corrigir e re-validar até passar.

## 3. Reload (não restart)

```bash
systemctl reload ssh
systemctl status ssh --no-pager | head -10
```

`reload` re-lê config sem matar conexões abertas. `restart` derruba e recria — evita-se em sshd justamente porque um config quebrado pode deixar você sem acesso.

## 4. Verificação

Em **outra janela do PowerShell** (não na sessão atual — pra não derrubar você caso algo dê errado):

```powershell
# Deve continuar funcionando (chave OK)
ssh vm-hostinger "echo 'key-only login OK'"

# Deve falhar com 'Permission denied (publickey)' — sem fallback pra senha
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password vm-hostinger "echo should not get here"
```

Esperado:
1. Primeira: `key-only login OK`.
2. Segunda: `Permission denied (publickey).` (NUNCA deve perguntar senha).

## Recovery: e se eu me trancar fora?

A Hostinger expõe console VNC/web no painel da VPS — login por usuário/senha lá ainda funciona (não passa por sshd). Logar como root via console, remover `/etc/ssh/sshd_config.d/10-datajoin-hardening.conf`, `systemctl reload ssh`, e investigar.

## Próximos passos

- `04-firewall.md` — `ufw` deny-default + libera SSH; `fail2ban` pra brute-force.
- Considerar criar usuário não-root (`03b-`) quando fizer sentido.
