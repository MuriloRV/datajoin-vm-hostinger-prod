# 02 — System baseline

Configura o básico do SO antes de qualquer software (Docker, k3s, etc): desativa regenerações automáticas do cloud-init, hostname, timezone, locale, atualização de pacotes e ferramentas essenciais. Tudo idempotente — pode rodar de novo sem efeito colateral.

Roda **na VM** (`ssh vm-hostinger`).

## 1. Desabilitar cloud-init managing de hostname e /etc/hosts

A imagem da Hostinger vem com `cloud-init` ativo + defaults perigosos: `preserve_hostname: false` (reseta hostname no boot pra valor da metadata) e `manage_etc_hosts: true` (regenera `/etc/hosts` do template). Isso desfaz qualquer mudança nossa.

Override via drop-in (NÃO editar `/etc/cloud/cloud.cfg` direto — pode ser sobrescrito em upgrade do pacote):

```bash
cat > /etc/cloud/cloud.cfg.d/99-datajoin.cfg <<'EOF'
# datajoin: assumimos ownership total do hostname e /etc/hosts deste VM.
# cloud-init não deve regenerar nenhum dos dois.
preserve_hostname: true
manage_etc_hosts: false
EOF
```

## 2. Hostname

```bash
hostnamectl set-hostname datajoin-app-prod
```

Atualizar `/etc/hosts` pra mapear o novo hostname pro loopback (alguns programas reclamam se isso falta — ex: `sudo` warns, kubelet refuses). Apaga **todas** as linhas `127.0.1.1` (o template da Hostinger pode vir com duplicatas) e reescreve uma só — idempotente:

```bash
sed -i '/^127\.0\.1\.1/d' /etc/hosts
echo -e "127.0.1.1\tdatajoin-app-prod" >> /etc/hosts
```

## 3. Timezone

```bash
timedatectl set-timezone America/Sao_Paulo
```

Verificar:
```bash
timedatectl status | grep "Time zone"
```

## 4. Locale

Manter `LANG=en_US.UTF-8` no servidor (logs/output em inglês pra consistência com docs e troubleshooting), mas gerar `pt_BR.UTF-8` pra ficar disponível pra apps que precisam de formatação BR (datas, números).

```bash
locale-gen en_US.UTF-8 pt_BR.UTF-8
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

## 5. Atualizar pacotes

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get upgrade -y
apt-get autoremove -y
```

`DEBIAN_FRONTEND=noninteractive` evita prompts (ex: editor de configs, dialogs do `needrestart`).

## 6. Ferramentas essenciais

Pacotes que sempre vão fazer falta. `gnupg`, `lsb-release`, `ca-certificates` são pré-requisitos pros próximos passos (Docker repo no `05`).

```bash
apt-get install -y \
  curl wget \
  htop tmux \
  git vim \
  jq ncdu \
  net-tools dnsutils \
  unzip \
  ca-certificates gnupg lsb-release \
  apt-transport-https software-properties-common
```

## 7. Verificação

```bash
hostnamectl
timedatectl status
locale
df -h /
```

Esperado: hostname `datajoin-app-prod`, timezone `America/Sao_Paulo`, LANG `en_US.UTF-8`, e a partição raiz com espaço suficiente.

## Notas

- **Reboot opcional:** se o `apt upgrade` atualizou kernel/libc, pode valer um `reboot` pra carregar o novo kernel. `cat /var/run/reboot-required 2>/dev/null && echo "reboot recomendado"` indica.
- **Não foi criado usuário não-root.** Acesso fica via `root` por chave SSH. Quando quiser segregar, criar `03b-admin-user.md` ou similar.
