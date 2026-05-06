# cloudflared

Container que mantém o Cloudflare Tunnel aberto desta VM até a Cloudflare. Tráfego HTTP(S) público dos apps entra por aqui.

Doc completo do passo: [`../docs/06-cloudflare-tunnel.md`](../docs/06-cloudflare-tunnel.md).

## Deploy resumido

Na VM (`/opt/datajoin/cloudflared/`):

```bash
# 1. Network compartilhada (ainda não criada nesta etapa, ver doc)
docker network create dj_network 2>/dev/null || true

# 2. .env com o token (NUNCA commit)
cp .env.example .env
$EDITOR .env   # cola o TUNNEL_TOKEN

# 3. Sobe
docker compose up -d
docker compose logs --tail 30
```
