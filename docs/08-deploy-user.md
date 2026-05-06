# 08 — Deploy user pra CI/CD

Cria usuário **não-root** `deploy` na VM com acesso a Docker, recebe a chave SSH do GitHub Actions, e prepara `/opt/datajoin/app/`. **Nada além disso roda manualmente** — o workflow do GHA materializa `.env`, `compose.yml`, faz `docker login`, pull, migrate, up.

Roda **na VM** (`ssh vm-hostinger` — root).

## Princípio: zero passos manuais recorrentes

A regra do workspace é "tudo via repo, deploy 100% automatizado". Por isso este doc cobre **só o que é estritamente one-time per VM** (provisioning). Qualquer operação que precise repetir (sync de compose, login no registry, edição de env) é responsabilidade do GHA workflow no repo `datajoin-app`.

## 1. Criar o user `deploy` (na VM)

```bash
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
install -d -o deploy -g deploy -m 700 /home/deploy/.ssh
touch /home/deploy/.ssh/authorized_keys
chown deploy:deploy /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys
```

User no grupo `docker` (executa containers sem sudo). Sem `sudoers` — atacante que rouba a CI key não consegue escalar pra root.

## 2. Diretório do app

```bash
install -d -o deploy -g deploy -m 755 /opt/datajoin
install -d -o deploy -g deploy -m 755 /opt/datajoin/app
install -d -o deploy -g deploy -m 755 /opt/datajoin/app/docker/postgres
```

`/opt/datajoin/app/` vai receber `compose.yml`, `.env`, `docker/postgres/init.sh` — todos materializados pelo GHA workflow a cada deploy. Não tem nada committado aqui no repo da VM (são artefatos do `datajoin-app`).

## 3. SSH key do CI

**Geração na workstation (one-time):**
```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_ci_deploy" -C "ci-deploy-datajoin-app" -N '""'
```

**Adicionar pública na VM (sem `command=` — workflow precisa flexibilidade pra rodar comandos arbitrários como deploy user):**
```powershell
$pub = Get-Content "$env:USERPROFILE\.ssh\id_ed25519_ci_deploy.pub" -Raw
ssh vm-hostinger "echo '$pub' | tee /home/deploy/.ssh/authorized_keys && chown deploy:deploy /home/deploy/.ssh/authorized_keys && chmod 600 /home/deploy/.ssh/authorized_keys"
```

## 4. GitHub Secrets (no repo `datajoin-app`)

`Settings → Secrets and variables → Actions → Secrets`. Necessários:

| Secret | Valor |
|---|---|
| `DEPLOY_SSH_KEY` | conteúdo de `~/.ssh/id_ed25519_ci_deploy` (privada) |
| `VM_HOST` | IP público da VM |
| `VM_HOST_KEY` | output de `ssh-keyscan -t ed25519 <VM_IP>` (cola a linha completa) |
| `APP_DOTENV` | conteúdo COMPLETO do `.env.prod` da app (preenchido a partir de `datajoin-app/.env.prod.example` com secrets reais) |

`APP_DOTENV` substitui o `.env` editado manualmente. Workflow cat'a esse secret na VM em `/opt/datajoin/app/.env` antes do `docker compose up`. Pra rotacionar uma senha: edita o secret no GH, próximo push redeploya com a senha nova.

GitHub Variables (não-secrets):

| Variable | Valor |
|---|---|
| `NEXT_PUBLIC_API_URL` | URL pública da API (ex: `https://api.<DOMAIN>`) — baked no bundle do portal em build time |

## Recovery / debug

- **CI deploy falhando**: `ssh vm-hostinger "tail -f /var/log/auth.log"` no momento do deploy pra ver SSH logs. `journalctl -u docker` pra docker daemon. `cd /opt/datajoin/app && docker compose logs` pra logs do app.
- **Key comprometida**: gerar nova `id_ed25519_ci_deploy_v2`, atualizar GH secret `DEPLOY_SSH_KEY` + sobrescrever `/home/deploy/.ssh/authorized_keys` na VM.
- **Wrapper antigo**: este doc na versão original tinha um wrapper `/usr/local/bin/datajoin-app-deploy.sh` com `command="..."` no authorized_keys. Foi descartado em favor de SSH livre + workflow flexível. Se algum cleanup ficou: `rm -f /usr/local/bin/datajoin-app-deploy.sh`.
