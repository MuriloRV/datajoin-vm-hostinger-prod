# 08 — Deploy user + CI SSH key

Cria usuário **não-root** `deploy` na VM, com acesso a Docker via grupo, SSH key dedicada pro GitHub Actions, e diretório `/opt/datajoin/app/` preparado pra receber o compose de produção do `datajoin-app`.

Roda **na VM** (`ssh vm-hostinger` — root).

## Por que user `deploy` separado?

CI/CD precisa de uma conta na VM pra fazer `docker compose pull && up -d` autonomamente. Opções:

1. **Reutilizar root** — qualquer comprometimento do GitHub Actions (supply chain, key leak) vira root na VM. **Não recomendado.**
2. **User `deploy` no grupo `docker`** — comprometimento limita-se a docker (ainda potente, mas sem mexer em SSH/sshd, sysctl, ufw, etc).
3. **User `deploy` + sudo só pra script wrapper** — máxima restrição. Adotamos isso.

**Defesa em camadas:**
- User dedicado (não root)
- Membro do grupo `docker` (executa containers)
- SSH key específica do CI, **forçada via `command="..."`** no `authorized_keys` a rodar **apenas** o wrapper de deploy `/usr/local/bin/datajoin-app-deploy.sh`. Mesmo se a key vazar, atacante só consegue triggar deploy — não abre shell.

## 1. Criar o user `deploy` (na VM, como root)

```bash
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
install -d -o deploy -g deploy -m 700 /home/deploy/.ssh
touch /home/deploy/.ssh/authorized_keys
chown deploy:deploy /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys
```

## 2. Diretório do app

```bash
install -d -o deploy -g deploy -m 755 /opt/datajoin
install -d -o deploy -g deploy -m 755 /opt/datajoin/app
install -d -o deploy -g deploy -m 755 /opt/datajoin/app/docker/postgres
```

## 3. Wrapper de deploy

Script único que CI invoca via SSH. Idempotente, falha limpa em qualquer erro (set -e), loga tudo (set -x).

```bash
cat > /usr/local/bin/datajoin-app-deploy.sh <<'EOF'
#!/bin/bash
# Wrapper de deploy do datajoin-app, invocado pelo GitHub Actions via SSH.
# Forcado via command="..." no authorized_keys do user deploy — o atacante
# que rouba a CI key SO consegue triggar este script.

set -euo pipefail
exec 2>&1   # stderr -> stdout (CI captura tudo)

cd /opt/datajoin/app

echo "==> docker compose pull"
docker compose pull

echo "==> alembic upgrade head"
docker compose run --rm api alembic upgrade head

echo "==> docker compose up -d --remove-orphans"
docker compose up -d --remove-orphans

echo "==> status final"
docker compose ps

echo "==> deploy OK"
EOF
chmod 755 /usr/local/bin/datajoin-app-deploy.sh
```

## 4. SSH key do CI (gera na **workstation**, NÃO na VM)

Em **PowerShell na sua workstation**:

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_ci_deploy" -C "ci-deploy-datajoin-app" -N '""'
```

`-N '""'` = sem passphrase (CI precisa não-interativa). Gera 2 arquivos:
- `~/.ssh/id_ed25519_ci_deploy` (privada — vai pro GitHub Secrets)
- `~/.ssh/id_ed25519_ci_deploy.pub` (pública — vai pra VM)

## 5. Autorizar a chave pública na VM (com restrição)

Pública vai pro `/home/deploy/.ssh/authorized_keys`, **com prefixo de restrição** que força a key a só rodar o deploy wrapper:

```powershell
$pub = Get-Content "$env:USERPROFILE\.ssh\id_ed25519_ci_deploy.pub" -Raw
$entry = "command=`"/usr/local/bin/datajoin-app-deploy.sh`",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty $pub"
ssh vm-hostinger "echo '$entry' >> /home/deploy/.ssh/authorized_keys && chown deploy:deploy /home/deploy/.ssh/authorized_keys && chmod 600 /home/deploy/.ssh/authorized_keys"
```

(Roda como root via alias `vm-hostinger`; o cara escreve em `/home/deploy/`.)

Verifica:
```bash
ssh vm-hostinger "tail -1 /home/deploy/.ssh/authorized_keys"
```

A linha deve começar com `command="/usr/local/bin/datajoin-app-deploy.sh"`.

## 6. Subir a chave privada como GitHub Secret

**NÃO** colar conteúdo da chave em chat ou em qualquer lugar logado. Fluxo manual:

1. Abre `https://github.com/MuriloRV/datajoin-app/settings/secrets/actions`.
2. Clica **New repository secret**.
3. **Name**: `DEPLOY_SSH_KEY`.
4. **Value**: cola TODO o conteúdo de `~/.ssh/id_ed25519_ci_deploy` (incluindo as linhas `-----BEGIN OPENSSH PRIVATE KEY-----` e `-----END...-----`). Pra abrir no notepad: `notepad $env:USERPROFILE\.ssh\id_ed25519_ci_deploy`.
5. **Add secret**.

Outros secrets que o workflow vai precisar (criar do mesmo jeito):

| Secret | Valor |
|---|---|
| `DEPLOY_SSH_KEY` | conteúdo da chave privada (passo acima) |
| `VM_HOST` | IP público da VM (`<VM_IP>`) |
| `VM_HOST_KEY` | output de `ssh-keyscan -t ed25519 <VM_IP>` na sua workstation — fingerprint da VM, evita TOFU prompt no runner |

## 7. (Opcional mas recomendado) Apagar privada da workstation

Depois que a key tá no GH secret, ela não precisa mais existir localmente — CI roda do runner. Se vazar do workstation, é vetor de ataque a mais.

```powershell
Remove-Item "$env:USERPROFILE\.ssh\id_ed25519_ci_deploy"
# Mantem a publica (.pub) — nao tem valor secreto, util pra debug
```

## 8. Smoke test (sem CI ainda)

Antes de configurar o GHA, valida que o setup todo funciona via SSH manual:

```powershell
# Tem que copiar compose.prod.yml + .env real + init.sh pra VM antes (ver passo 9).
ssh deploy@<VM_IP>
# Esperado: o wrapper roda, docker compose pull baixa imagens, falha porque imagens
# ainda nao existem no ghcr.io. Ate aqui: setup OK, falta CI publicar imagens.
```

## 9. Copiar artefatos iniciais pra VM

Antes do primeiro deploy do CI, alguns arquivos precisam estar na VM (CI não copia, só pulla imagens):

```bash
# /opt/datajoin/app/compose.yml — copiar do datajoin-app/compose.prod.yml (renomeado)
# /opt/datajoin/app/.env — preencher a partir de .env.prod.example
# /opt/datajoin/app/docker/postgres/init.sh — bind-mount referenciado pelo compose
```

(Esses 3 arquivos ficam só na VM, não no repo do datajoin-vm-hostinger-prod, porque pertencem ao app — se mudar a estrutura do app, vem do `datajoin-app` repo.)

## Recovery

- **CI deploy falhando**: SSH como root (sua chave normal), `tail -100 /home/deploy/.ssh/authorized_keys` pra confirmar a entry, `cat /usr/local/bin/datajoin-app-deploy.sh` pra confirmar o script, `docker compose -f /opt/datajoin/app/compose.yml ps` pra ver containers.
- **Key comprometida**: rotacionar — gerar nova `id_ed25519_ci_deploy_v2`, atualizar GH secret + authorized_keys, deletar a antiga linha do `authorized_keys` na VM.
