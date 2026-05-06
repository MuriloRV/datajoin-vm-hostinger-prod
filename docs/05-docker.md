# 05 — Docker + Compose

Instala Docker Engine + Compose v2 (plugin oficial) via repositório oficial da Docker. Configura `daemon.json` com log rotation e `live-restore` (containers sobrevivem a restart do daemon durante upgrades).

Roda **na VM** (`ssh vm-hostinger`).

## Por que repo oficial e não `docker.io` do Ubuntu

- Versões mais novas, suporte direto.
- Plugin Compose v2 (`docker compose ...`, sem hífen) — `docker-compose` (com hífen, v1 em Python) está EOL desde 2023.
- Mesma fonte que a doc oficial — menos pegadinhas.

## 1. Adicionar repo oficial Docker

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list

apt-get update
```

## 2. Instalar engine + plugins

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## 3. Configurar `/etc/docker/daemon.json`

Defaults sensatos pra produção: rotação de logs (sem isso, um container barulhento enche o disco) e `live-restore` (containers continuam rodando quando o daemon reinicia em upgrade).

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true
}
EOF
```

Aplicar:
```bash
systemctl restart docker
systemctl enable docker
```

## 4. Verificação

```bash
docker version
docker compose version
docker info | grep -E '(Live Restore|Logging Driver|Docker Root Dir|Kernel)'
docker run --rm hello-world
```

Esperado:
- `docker version`: client + server respondendo, mesma major version.
- `docker compose version`: v2.x.
- `docker info`: `Live Restore Enabled: true`, `Logging Driver: json-file`.
- `hello-world`: pull + run + mensagem da Docker.

## Gotcha: ufw vs Docker port publishing

Docker manipula iptables direto (chains `DOCKER`, `DOCKER-USER`). Quando você publica porta com `-p 80:80`, o tráfego entra via PREROUTING/FORWARD e **bypassa** as regras INPUT do ufw. Ou seja, `ufw deny 80/tcp` NÃO impede um container publicado de ser acessível.

Mitigação no nosso setup: **não publicar portas ao host**. Tudo vai por Cloudflare Tunnel (passo 06), que faz outbound da VM e expõe os serviços via Cloudflare. Containers ficam só nas docker networks internas — nada de `-p 80:80` no compose de produção.

Se algum dia publicar porta, opções: (a) bind explícito a `127.0.0.1:PORT:PORT` (acessível só pela própria VM), (b) regra customizada no chain `DOCKER-USER`.

## Limpeza opcional

A imagem da Hostinger pode ter algum docker antigo pré-instalado (`docker.io`, `docker-doc`, `docker-compose`, etc). Se aparecer conflito, remover ANTES de instalar:
```bash
for p in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  apt-get remove -y "$p" 2>/dev/null || true
done
```
