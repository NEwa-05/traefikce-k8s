# Traefik OSS demo in K8s


## Set environment variable

```bash
source .env
```

## Change ingress domain

```bash
for i in $(grep -Rl '${CLUSTERNAME}'); do gsed -i 's/${CLUSTERNAME}/'$CLUSTERNAME'/g' $i; done
```

## Set Traefik Helm repo

```bash
helm repo add traefik https://traefik.github.io/charts
```

### Update Hub Helm repo for new releases

```bash
helm repo update
```

## deploy Traefik OSS from helm chart

```bash
helm upgrade --install traefik traefik/traefik -f values.yaml -n traefik --create-namespace
```

## Wait for cloud provider LB address

```bash
kubectl -n traefik get svc -w
```

## Set env-on-demand DNS entry

```bash
curl --location --request POST "https://cf.infra.traefiklabs.tech/dns/env-on-demand" \
--header "X-TraefikLabs-User: ${CLUSTERNAME}" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${DNS_TOKEN}" \
--data-raw "{\"Value\": \"$(kubectl get service traefik -n traefik --no-headers | awk {'print $4'})\"}"
```

## Deploy Traefik dashboard

```bash
kubectl apply -f dashboard.yaml
```

## Deploy whoami app

```bash
kubectl apply -f whoami
```

## Create Observability namespace

```bash
kubectl create ns observability
```

## Create custom dashboard for Traefik metrics

```bash
kubectl create configmap grafana-traefik-dashboard \
  --from-file=tools/observability/dashboard-traefik.json \
  -o yaml --dry-run=client -n observability | kubectl apply -f -
```

## Deploy Loki to ship logs

```bash
helm upgrade --install loki grafana/loki-stack \
  -f tools/observability/loki-values.yaml \
  --namespace=observability \
  --create-namespace
```

## Deploy Prometheus Stack with Grafana

```bash
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  -f tools/observability/prom-values.yaml \
  --namespace=observability \
  --create-namespace
```

### set Grafana UI ingress

```bash
kubectl apply -f tools/observability/grafana-ingress.yaml
```
