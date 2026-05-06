# datajoin-vm-hostinger-prod

Configuração reproduzível da VM Hostinger de produção da datajoin. Tudo que está aplicado na VM precisa estar versionado aqui — **nenhuma ação direta na VM sem código equivalente neste repo**.

Configs específicas do app/airflow não pertencem aqui (vivem em `datajoin-app/` e `datajoin-airflow/`). O escopo deste repo é a base da máquina: SSH, hardening, Docker, Cloudflare Tunnel, k3s, etc.

## VM

| Campo | Valor |
|---|---|
| Provedor | Hostinger |
| IP público | `<VM_IP>` (valor real fora do repo, no `~/.ssh/config` da workstation) |
| OS | Ubuntu 22.04.5 LTS (Jammy Jellyfish) |
| Kernel | 5.15.0-174-generic |
| CPU / RAM | 4 cores / 15 GiB |
| Disco | 194 GB (~2 GB usados em VM limpa) |
| Swap | nenhum configurado (avaliar conforme workload) |
| Acesso inicial | `ssh vm-hostinger` (root, autenticação por chave) |

## Passos

A configuração é numerada na ordem de execução. Ao executar um passo, atualizar este repo primeiro se houver desvio.

| # | Passo | Status | Doc |
|---|---|---|---|
| 01 | Gerar chave SSH local + autorizar na VM + configurar `~/.ssh/config` | ✓ feito | [`docs/01-ssh-key.md`](docs/01-ssh-key.md) |
| 02 | System baseline (cloud-init override, hostname, TZ, locale, updates, essenciais) | ✓ feito | [`docs/02-system-baseline.md`](docs/02-system-baseline.md) |
| 03 | SSH hardening (key-only, sem senha — mantém root) | ✓ feito | [`docs/03-ssh-hardening.md`](docs/03-ssh-hardening.md) |
| 04 | Firewall (`ufw` + `fail2ban`) | ✓ feito | [`docs/04-firewall.md`](docs/04-firewall.md) |
| 05 | Docker + Compose | ✓ feito | [`docs/05-docker.md`](docs/05-docker.md) |
| 06 | Cloudflare Tunnel | ✓ feito (público hostnames pendentes p/ apps) | [`docs/06-cloudflare-tunnel.md`](docs/06-cloudflare-tunnel.md) |
| 07 | k3s + Helm | ✓ feito | [`docs/07-k3s-helm.md`](docs/07-k3s-helm.md) |
