# 07 — k3s + Helm

Instala k3s (distribuição leve de Kubernetes da Rancher, single binary) + Helm 3 pra deployar o `datajoin-airflow` via Helm chart no passo seguinte. Single-node cluster — k3s é desenhado pra rodar em hardware modesto, footprint ~500–700 MiB RAM.

Roda **na VM** (`ssh vm-hostinger`).

## Decisões de design

**`--disable traefik`** — k3s vem com Traefik como ingress controller default. Não usamos ingress: tráfego HTTP público entra via Cloudflare Tunnel (passo 06), que termina TLS e roteia direto pros services k3s via NodePort/ClusterIP. Traefik seria uma camada extra inútil consumindo RAM.

**`--disable servicelb`** — service LoadBalancer (klipper-lb) também não faz sentido em single-node sem cloud LB; expomos via NodePort quando precisar.

**`--write-kubeconfig-mode 644`** — kubeconfig (`/etc/rancher/k3s/k3s.yaml`) world-readable. Estamos em single-user (root), mas o mode 644 simplifica integração com tools.

**`--node-name datajoin-app-prod`** — bate com o hostname; node aparece com nome correto em `kubectl get nodes`.

**Datastore: SQLite (default)** — k3s usa sqlite embedded pra single-node, suficiente. Mudaria pra etcd ou DB externo só pra HA.

## 1. Instalar k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --disable=servicelb --write-kubeconfig-mode=644 --node-name=datajoin-app-prod" sh -
```

Esse script:
- Baixa o binário k3s pra `/usr/local/bin/k3s`
- Instala symlinks (`kubectl`, `crictl`, `ctr` apontando pro k3s)
- Cria systemd unit `k3s.service`, habilita e inicia
- Escreve kubeconfig em `/etc/rancher/k3s/k3s.yaml`

## 2. Configurar `KUBECONFIG`

Pra `kubectl` funcionar em qualquer shell sem `--kubeconfig=...`:

```bash
cat > /etc/profile.d/k3s.sh <<'EOF'
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
EOF
chmod +x /etc/profile.d/k3s.sh
# carregar na sessao atual
. /etc/profile.d/k3s.sh
```

## 3. Liberar tráfego de pods/services no ufw

k3s usa flannel CNI: pods em `10.42.0.0/16`, services em `10.43.0.0/16`. ufw bloqueia 2 coisas que precisamos liberar:

1. **Tráfego INPUT** vindo dessas subnets pra serviços rodando no host (ex: kube-apiserver no 6443).
2. **Tráfego FORWARD entre interfaces** — `flannel.1` pra `cni0` e vice-versa. Default do Ubuntu é `deny (routed)`, e isso impede pod→service→pod (a service VIP é virtual, sai por uma interface e volta por outra após DNAT/SNAT).

```bash
# 3a. Pod e service CIDRs como source confiável
ufw allow from 10.42.0.0/16 to any comment 'k3s pods CIDR'
ufw allow from 10.43.0.0/16 to any comment 'k3s services CIDR'

# 3b. Permitir FORWARD (default routed = allow)
ufw default allow routed

ufw reload
ufw status verbose | head -5
```

A linha de Default deve mostrar `deny (incoming), allow (outgoing), allow (routed)`. Sem o `allow (routed)`, smoke tests passam o `kubectl wait` mas qualquer ClusterIP de service dá timeout.

Segurança: `allow routed` permite forwarding entre interfaces da VM, mas `default deny incoming` continua valendo na entrada — nenhum tráfego externo é forwarded pra dentro da rede de pods (a VM não é um roteador externo).

## 4. Verificar cluster

```bash
kubectl get nodes
kubectl get pods -A
kubectl version --short
```

Esperado:
- `kubectl get nodes` mostra `datajoin-app-prod` com status `Ready`.
- `kubectl get pods -A` mostra pods em `kube-system` (`coredns`, `local-path-provisioner`, `metrics-server`) `Running`. **Não** deve aparecer `traefik` nem `svclb-*` (foram desabilitados).
- `kubectl version --short` mostra Client e Server na mesma versão.

## 5. Smoke test

```bash
kubectl run smoke-test --image=nginx:alpine --restart=Never
kubectl wait --for=condition=Ready pod/smoke-test --timeout=60s

# DNS interno via CoreDNS
kubectl exec smoke-test -- nslookup kubernetes.default.svc.cluster.local
# Routing pod -> service VIP (deve dar 401 Unauthorized — pod sem token —, NÃO timeout)
kubectl exec smoke-test -- wget -qO- --timeout=5 https://kubernetes.default.svc --no-check-certificate

kubectl delete pod smoke-test
```

Se DNS resolver e o wget retornar 401 (não timeout), tanto CoreDNS quanto routing de service VIP via flannel/iptables tão OK.

## 6. Instalar Helm 3

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Esperado: Helm v3.x.x e BuildInfo apontando GoVersion recente.

## 7. Verificação final

```bash
kubectl get nodes
helm list -A
```

Cluster Ready, sem helm releases ainda (vamos criar o do Airflow no próximo passo).

## Recovery / troubleshooting

- **k3s não inicia**: `journalctl -u k3s -f` pra logs. Comum: faltando memória, conflito de CIDR com docker bridge (default docker `172.17.0.0/16` não conflita com k3s `10.42`/`10.43`).
- **kubectl "connection refused"**: serviço k3s caiu. `systemctl status k3s` e `systemctl restart k3s`.
- **Pods stuck em ContainerCreating**: provável CNI quebrou — checar `kubectl get pods -n kube-system` e logs do flannel.
- **Desinstalar k3s** (se precisar resetar): `/usr/local/bin/k3s-uninstall.sh` (criado pelo installer).

## Próximo passo

`08 — datajoin-app deploy` (compose) e/ou `09 — datajoin-airflow Helm chart` (k3s). Os dois precisam acertar credenciais do Postgres + DNS abstrato (`db.<DOMAIN>`).
