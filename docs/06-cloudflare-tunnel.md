# 06 — Cloudflare Tunnel

Conecta a VM à internet via **Cloudflare Tunnel** sem expor portas TCP/UDP. O `cloudflared` roda como container na VM, abre conexão **outbound** persistente até a Cloudflare; o tráfego de entrada (HTTPS dos seus apps) é aceito por edge da Cloudflare, encriptado, encaminhado pelo túnel até a VM, e roteado pra um container específico via "public hostname".

**Vantagens:**
- Não precisa abrir 80/443 no `ufw` (a internet pública nunca alcança a VM diretamente).
- TLS terminado na Cloudflare — não precisa Let's Encrypt na VM.
- Cloudflare Access (Zero Trust) permite gating por SSO/email/IP em qualquer hostname com 2 cliques.
- Se trocar de VM, basta levar o token e subir o `cloudflared` lá; nenhuma config de DNS muda.

**Pré-requisitos já satisfeitos:**
- Conta Cloudflare ativa.
- Domínio adicionado e com nameservers da Cloudflare (DNS proxiado, "laranja").

---

## 0. Ativar Cloudflare Zero Trust (one-time, painel)

Zero Trust é o produto que hospeda o Tunnel + Access. Free plan suporta 50 usuários — mais que suficiente.

**0.1.** Login em `https://dash.cloudflare.com`.

**0.2.** Na sidebar esquerda, clicar em **Zero Trust**. (Se não aparece, ir em "Account home" e procurar lá. O ícone é geralmente um escudo.)

**0.3.** Você cai em `https://one.dash.cloudflare.com` (subdomínio do Zero Trust). Como é primeira vez, abre um wizard de onboarding:

   **a)** **Choose a team name**: vira o subdomínio `<team>.cloudflareaccess.com`, que é a URL do portal de login Access. Sugestão: `datajoin` → `datajoin.cloudflareaccess.com`. Ele é **fixo depois de criado** — escolhe um nome neutro/duradouro (não `datajoin-test`).

   **b)** **Choose a plan**: selecionar **Free**. Limite: 50 usuários, 24h log retention.

   **c)** **Payment method**: a Cloudflare exige cartão na conta mesmo no Free plan (mudança de 2023+ pra prevenir abuso). Não cobra nada se você ficar dentro do free tier. Adicionar cartão.

   **d)** Finalizar setup. Você cai no dashboard Zero Trust.

---

## 1. Criar o Tunnel (painel)

**1.1.** Dashboard Zero Trust → sidebar **Networks** → **Tunnels**.

**1.2.** Botão **Create a tunnel** (canto superior direito).

**1.3.** **Connector type**: escolher **Cloudflared** (NÃO WARP Connector). Click "Next".

**1.4.** **Name your tunnel**: `datajoin-app-prod`. (Convenção: nome da VM. Se um dia tiver staging, criar `datajoin-app-staging` separado — tunnels não custam nada.)

**1.5.** **Save tunnel**. Na próxima tela aparece "Install and run a connector".

**1.6.** Clicar na aba **Docker**. Aparece um comando longo tipo:
```
docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token eyJhIj...<MUITAS LETRAS>...QzIifQ==
```

**Copiar o token** (a string longa depois de `--token`, sem incluir `--token`). É **isso** que vai pra `TUNNEL_TOKEN` na VM.

⚠️ **Tratamento do token:**
- É **credencial** de longo prazo (não rotaciona automaticamente). Quem tiver o token pode subir um conector e impersonar esse tunnel.
- **NUNCA commitar** no repo (o `.gitignore` deste repo já bloqueia `.env`).
- Vive só em `/opt/datajoin/cloudflared/.env` na VM.

**1.7.** **NÃO clicar "Next"** ainda. Deixa essa tela aberta — a próxima etapa só é satisfeita quando o `cloudflared` na VM se reportar à Cloudflare. Volta aqui depois.

---

## 2. Estrutura no repo

Toda a configuração do `cloudflared` (compose, env example) vive em [`../cloudflared/`](../cloudflared/) deste repo:

```
cloudflared/
├── compose.yml         # serviço cloudflared
├── .env.example        # template (TUNNEL_TOKEN=)
└── README.md           # instruções resumidas
```

A pasta `/opt/datajoin/cloudflared/` na VM é a **réplica** dessa estrutura mais o `.env` real (com o token). O `.env` real **nunca** entra no repo.

---

## 3. Deploy do `cloudflared` na VM

Caminho na VM: `/opt/datajoin/cloudflared/`. Padrão: serviços de infra ficam em `/opt/datajoin/<serviço>/`.

**3.1.** Criar a pasta e copiar o `compose.yml` (mesma versão do repo).

**3.2.** Criar o `.env` com o token copiado no passo 1.6:
```bash
cat > /opt/datajoin/cloudflared/.env <<EOF
TUNNEL_TOKEN=eyJhIj...QzIifQ==
EOF
chmod 600 /opt/datajoin/cloudflared/.env   # nao deixar world-readable
```

**3.3.** Subir o serviço:
```bash
cd /opt/datajoin/cloudflared
docker compose up -d
docker compose ps
docker compose logs --tail 30
```

Esperado nos logs:
- `Registered tunnel connection ...` (várias linhas, conector se conecta a 4 datacenters Cloudflare diferentes pra HA).
- `Connection ... registered with protocol: quic`.

---

## 4. Verificação no painel

**4.1.** Voltar à tela "Install and run a connector" (passo 1.7). Em cima da listagem de comandos, há um indicador "Connectors" que vira **verde com 1 conector ativo**. Click **Next**.

**4.2.** Próxima tela: **Public hostnames**. Pula por enquanto — vai configurar à medida que cada app subir. Pode clicar **Skip** ou setar uma rota dummy temporária. Click **Save tunnel**.

**4.3.** Lista de tunnels deve mostrar `datajoin-app-prod` com status **Healthy** (verde).

---

## 5. Configurar public hostnames (depois que apps subirem)

Para cada serviço público (portal, API, Airflow UI), criar uma route:

**5.1.** Dashboard Zero Trust → Networks → Tunnels → `datajoin-app-prod` → aba **Public Hostnames** → **Add a public hostname**.

**5.2.** Preencher:
- **Subdomain**: `app` (vai virar `app.<seu-dominio>`)
- **Domain**: o domínio adicionado na conta
- **Path**: vazio (ou `/api/*` etc se quiser sub-paths)
- **Service**:
  - **Type**: `HTTP`
  - **URL**: `portal:3000` — nome do container na docker network compartilhada + porta interna. **Não** colocar `localhost:3000` (cloudflared roda em container, `localhost` dentro dele é o próprio container).

**5.3.** Save. Cloudflare cria automaticamente o DNS record `app.<dominio>` apontando pro tunnel. Levam ~30s pra propagar.

**5.4.** Testar: `curl -I https://app.<dominio>`. Se retornar 200 ou 301, tunnel funcionando.

---

## 6. Networking entre cloudflared e os apps

`cloudflared` precisa estar na **mesma docker network** dos services que vai rotear, pra resolver `portal:3000` etc por DNS interno do Docker.

Padrão neste setup:
- Network compartilhada **externa**: `datajoin_net` (nome final a confirmar — combina com a usada nos composes do `datajoin-app` e `datajoin-airflow`, que hoje usam `dj_network`).
- `cloudflared/compose.yml` declara essa network como `external: true` e adiciona o container nela.
- Quando o `datajoin-app` subir o `portal` na mesma network, o cloudflared resolverá `portal` → IP interno do container.

A criação da network e a integração com os outros composes vai ser tratada quando começarmos a deployar o app (`08+`). Por ora, o `cloudflared` sobe sozinho na network e fica idle até ter target.

---

## Recovery / troubleshooting

- **Tunnel "unhealthy" no painel**: `docker compose logs cloudflared` na VM. Erros típicos: token errado (invalidado, expired, ou da conta errada) — gerar novo token.
- **502/503 em hostname**: cloudflared não consegue resolver/alcançar o `service:port` declarado. Checar se o container alvo tá up e na mesma docker network.
- **Vazou o token**: ir no painel → tunnel → "Refresh" o token (gera um novo, invalida o antigo). Atualizar `.env` na VM e `docker compose up -d` (recreate).
