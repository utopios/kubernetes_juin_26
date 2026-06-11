# Workshop Helm 3 — Stack de monitoring complète avec des charts publics

## Objectifs

À la fin, nous saurons :

* Installer une stack de monitoring complète (Prometheus, Grafana, Alertmanager) avec le chart public `kube-prometheus-stack`.
* Personnaliser une installation via un fichier `values.yaml` (sans jamais modifier le chart).
* Exposer les métriques d'une application avec un `ServiceMonitor`.
* Ajouter la collecte de logs avec Loki + Promtail et la brancher dans Grafana.
* Créer une règle d'alerte Prometheus via les CRDs du chart.
* Mettre à jour et désinstaller proprement une release.

## Pré-requis

* Un cluster Kubernetes fonctionnel (GKE, minikube, kind...).
* kubectl et Helm 3 installés et configurés.
* Connaissances Kubernetes et Helm de base (workshops 1 et 2).

---

## Architecture cible

```
┌─────────────────────────────────────────────────────────┐
│                  namespace: monitoring                  │
│                                                         │
│  ┌────────────┐   scrape   ┌──────────────────────────┐ │
│  │ Prometheus │◄───────────│ node-exporter (DaemonSet)│ │
│  │            │◄───────────│ kube-state-metrics       │ │
│  │            │◄───────────│ ServiceMonitor demo-app  │ │
│  └─────┬──────┘            └──────────────────────────┘ │
│        │ alertes                                        │
│  ┌─────▼────────┐          ┌─────────┐    ┌──────────┐  │
│  │ Alertmanager │          │  Loki   │◄───│ Promtail │  │
│  └──────────────┘          └────┬────┘    └──────────┘  │
│                                 │                       │
│  ┌──────────────────────────────▼─────┐                 │
│  │ Grafana (datasources: Prom + Loki) │                 │
│  └────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────┘
```

---

## Étape 0 — Vérification du cluster

```bash
kubectl get nodes
kubectl create namespace monitoring
```

---

## Étape 1 — Ajout des dépôts publics

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Explorons ce que proposent ces dépôts :

```bash
helm search repo prometheus-community/kube-prometheus-stack
helm search repo grafana/loki
```

Regardons les valeurs par défaut du chart principal (elles sont nombreuses, c'est normal — ce chart agrège Prometheus Operator, Grafana, Alertmanager, node-exporter et kube-state-metrics) :

```bash
helm show values prometheus-community/kube-prometheus-stack | head -100
helm show values prometheus-community/kube-prometheus-stack > /tmp/kps-defaults.yaml
```

> 💡 `kube-prometheus-stack` est un **chart parapluie** : il déclare Grafana, node-exporter, etc. comme dépendances. Exactement le mécanisme `dependencies:` vu au workshop 2, mais à l'échelle industrielle.

---

## Étape 2 — Fichier de valeurs personnalisé

Ne modifions **jamais** un chart public : on le pilote avec nos propres valeurs.

Créons `monitoring-values.yaml` :

```yaml
# monitoring-values.yaml
fullnameOverride: kps

grafana:
  adminPassword: "Workshop2026!"
  service:
    type: ClusterIP
  persistence:
    enabled: true
    size: 2Gi

prometheus:
  prometheusSpec:
    retention: 7d
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi
    # Permet à Prometheus de découvrir les ServiceMonitors
    # de TOUS les namespaces, pas seulement ceux de la release
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

alertmanager:
  alertmanagerSpec:
    resources:
      requests:
        cpu: 50m
        memory: 128Mi
```

> ⚠️ Le point important : `serviceMonitorSelectorNilUsesHelmValues: false`. Sans cette valeur, Prometheus ne scrape que les ServiceMonitors portant le label de la release Helm — un piège classique à l'étape 5.

---

## Étape 3 — Installation de la stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values monitoring-values.yaml \
  --version 65.x.x
```

> 💡 On fige la version majeure du chart (`--version`) : en production, une installation reproductible ne dépend jamais du « latest » du dépôt.

Observons ce qui a été déployé :

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get crds | grep monitoring.coreos.com
```

Nous devons voir :

* `kps-operator` : le Prometheus Operator, qui transforme les CRDs en configuration.
* `prometheus-kps-prometheus-0` : Prometheus lui-même (StatefulSet).
* `alertmanager-kps-alertmanager-0` : Alertmanager.
* `monitoring-grafana-...` : Grafana.
* `monitoring-prometheus-node-exporter-...` : un pod par nœud (DaemonSet).
* `monitoring-kube-state-metrics-...` : métriques des objets Kubernetes.

```bash
helm list -n monitoring
helm status monitoring -n monitoring
```

---

## Étape 4 — Accès aux interfaces

### Grafana

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Ouvrons [http://localhost:3000](http://localhost:3000) — login `admin` / `Workshop2026!`.

Le chart a déjà provisionné :

* La datasource Prometheus (Configuration → Data sources).
* Une trentaine de dashboards Kubernetes prêts à l'emploi (Dashboards → Browse) : explorez **Kubernetes / Compute Resources / Namespace (Pods)**.

### Prometheus

```bash
kubectl port-forward -n monitoring svc/kps-prometheus 9090:9090
```

Ouvrons [http://localhost:9090](http://localhost:9090) :

* **Status → Targets** : toutes les cibles scrapées.
* Testons une requête PromQL :

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="monitoring"}[5m])) by (pod)
```

---

## Étape 5 — Monitorer notre propre application

Déployons une application qui expose des métriques au format Prometheus.

Créons `demo-app.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: default
  labels:
    app: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: quay.io/brancz/prometheus-example-app:v0.5.0
          ports:
            - name: http
              containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: default
  labels:
    app: demo-app
spec:
  selector:
    app: demo-app
  ports:
    - name: http
      port: 8080
      targetPort: http
```

```bash
kubectl apply -f demo-app.yaml
```

Maintenant le `ServiceMonitor` — la CRD installée par le chart qui dit à Prometheus « scrape ce service » :

Créons `demo-app-servicemonitor.yaml` :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: demo-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: demo-app
  endpoints:
    - port: http        # nom du port DU SERVICE, pas le numéro
      path: /metrics
      interval: 15s
```

```bash
kubectl apply -f demo-app-servicemonitor.yaml
```

Vérifions (1 à 2 minutes de délai) dans Prometheus → **Status → Targets** : une cible `default/demo-app` doit apparaître avec 2 endpoints **UP**.

Générons du trafic et observons :

```bash
kubectl run curl --rm -it --image=curlimages/curl --restart=Never -- \
  sh -c 'for i in $(seq 1 50); do curl -s demo-app.default:8080 > /dev/null; done; echo done'
```

Dans Prometheus :

```promql
  rate(http_requests_total{job="demo-app"}[1m])
```

---

## Étape 6 — Les logs avec Loki + Promtail

Prometheus collecte les métriques ; pour les logs, ajoutons Loki (stockage) et Promtail (agent de collecte).

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.isDefault=false \
  --set promtail.enabled=true
```

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/instance=loki
```

Promtail tourne en DaemonSet : un pod par nœud, qui lit les logs de tous les conteneurs.

### Brancher Loki dans Grafana

Dans Grafana → **Connections → Data sources → Add data source → Loki** :

* URL : `http://loki.monitoring:3100`
* Save & test.

Puis **Explore**, datasource Loki, requête LogQL :

```logql
{namespace="default", app="demo-app"}
```

Nous avons maintenant métriques **et** logs dans la même interface.

> 💡 Alternative : la datasource Loki peut aussi être provisionnée par valeurs Helm (`grafana.additionalDataSources` dans `monitoring-values.yaml`) — c'est l'approche GitOps. La manipulation manuelle ici est volontaire, pour comprendre ce que les valeurs automatisent.

---

## Étape 7 — Créer une alerte

Le chart installe aussi la CRD `PrometheusRule`. Créons une alerte qui se déclenche si notre demo-app n'a plus de réplica disponible.

Créons `demo-app-alert.yaml` :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: demo-app-rules
  namespace: monitoring
  labels:
    release: monitoring   # requis pour être découvert par Prometheus
spec:
  groups:
    - name: demo-app
      rules:
        - alert: DemoAppDown
          expr: kube_deployment_status_replicas_available{deployment="demo-app"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "demo-app n'a plus aucun réplica disponible"
```

```bash
kubectl apply -f demo-app-alert.yaml
```

Testons en cassant l'application :

```bash
kubectl scale deployment demo-app --replicas=0
```

Après ~2 minutes, dans Prometheus → **Alerts** : `DemoAppDown` passe en **Pending** puis **Firing**, et apparaît dans Alertmanager :

```bash
kubectl port-forward -n monitoring svc/kps-alertmanager 9093:9093
```

Ouvrons [http://localhost:9093](http://localhost:9093).

Réparons :

```bash
kubectl scale deployment demo-app --replicas=2
```

---

## Étape 8 — Cycle de vie de la release

### Mise à jour de la configuration

Modifions la rétention dans `monitoring-values.yaml` (`retention: 15d`), puis :

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values monitoring-values.yaml \
  --version 65.x.x

helm history monitoring -n monitoring
```

### Rollback

```bash
helm rollback monitoring 1 -n monitoring
```

### Désinstallation propre

```bash
helm uninstall loki -n monitoring
helm uninstall monitoring -n monitoring
```

> ⚠️ Helm ne supprime **pas** les CRDs (`kubectl get crds | grep monitoring.coreos.com`) ni les PVCs. C'est volontaire : les CRDs sont partagées et les données précieuses. Pour tout nettoyer :

```bash
kubectl delete crds alertmanagerconfigs.monitoring.coreos.com alertmanagers.monitoring.coreos.com \
  podmonitors.monitoring.coreos.com probes.monitoring.coreos.com prometheusagents.monitoring.coreos.com \
  prometheuses.monitoring.coreos.com prometheusrules.monitoring.coreos.com \
  scrapeconfigs.monitoring.coreos.com servicemonitors.monitoring.coreos.com \
  thanosrulers.monitoring.coreos.com
kubectl delete pvc -n monitoring --all
kubectl delete namespace monitoring
kubectl delete -f demo-app.yaml -f demo-app-servicemonitor.yaml
```

