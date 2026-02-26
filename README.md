# Centralized Logging on Minikube (EFK) — Incident Log Search

I built a lightweight **centralized logging stack (EFK = Elasticsearch + Fluentd + Kibana)** on **Minikube** to collect logs from all pods and namespaces in one place, then **search + filter** quickly during incidents (instead of tailing logs pod-by-pod).

---

## Purpose

Enable a simple incident workflow on Kubernetes:

**Alert → open Kibana → search errors → filter by namespace/pod → confirm root cause → validate fix**

---

## Problem

During an incident, logs are often **scattered**:

- Logs live inside pods (pods restart, logs rotate)
- Many namespaces/services make it hard to know where to start
- `kubectl logs` is useful, but incident response needs:
  - full-text **search**
  - **time-based** filtering
  - correlation across services
  - quick pattern detection (`error`, `timeout`, `500`)

**Bottom line:** I need one place to store and search logs across the cluster.

---

## Solution

I deployed an **EFK pipeline** on Minikube:

- **Fluentd (DaemonSet)** collects node/pod logs across the cluster
- Fluentd forwards logs to **Elasticsearch** (storage + indexing)
- **Kibana** reads from Elasticsearch so I can:
  - search logs by keywords
  - filter by namespace/pod/container
  - zoom into a time range during the incident

---

## Architecture

![EFK Architecture](screenshots/architecture.png)

This diagram shows logs from all namespaces flowing into Fluentd on each node, then into Elasticsearch for indexing and storage, with Kibana used for searching and filtering by time, namespace, and pod.

---

## Prerequisites

- Minikube installed
- kubectl installed
- Enough resources for Elasticsearch (**recommended:** 4 CPU / 8GB RAM)

---

## Repo Structure

```text
centralized-logging-efk/
├── README.md
├── elasticsearch.yaml
├── kibana.yaml
├── fluentd-configmap.yaml
├── fluentd-daemonset.yaml
├── demo-app.yaml
└── screenshots/
    ├── architecture.png
    ├── 01-efk-pods-running.png
    ├── 02-elasticsearch-service.png
    ├── 03-fluentd-daemonset.png
    ├── 04-demo-logs.png
    ├── 05-kibana-running.png
    ├── 06-index-pattern.png
    ├── 07-kibana-error-search.png
    └── 08-filter-by-namespace.png
````

---

## Step-by-step CLI (with screenshots)

> Create screenshots folder (once):

```bash
mkdir -p screenshots
```

### 1) Start Minikube (allocate enough resources)

```bash
minikube start --cpus=4 --memory=8192
kubectl get nodes -o wide
```

📸 `screenshots/00-minikube-node-ready.png` 
Should show: node is `Ready`.

---

### 2) Create a namespace for logging

```bash
kubectl create namespace logging
kubectl get ns
```

📸 `screenshots/00-logging-namespace.png` 
Should show: `logging` namespace exists.

---

### 3) Deploy Elasticsearch (single-node for demo)

```bash
kubectl apply -n logging -f elasticsearch.yaml
kubectl get pods -n logging -w
```

📸 `screenshots/01-efk-pods-running.png`
![EFK Pods Running](screenshots/01-efk-pods-running.png)

Validate Elasticsearch service:

```bash
kubectl get svc -n logging
kubectl get endpoints -n logging
kubectl logs -n logging deploy/elasticsearch --tail=50
```

📸 `screenshots/02-elasticsearch-service.png`
![Elasticsearch Service](screenshots/02-elasticsearch-service.png)

---

### 4) Deploy Kibana

```bash
kubectl apply -n logging -f kibana.yaml
kubectl get pods -n logging -w
kubectl get svc -n logging
```

📸 `screenshots/05-kibana-running.png`
![Kibana Running](screenshots/05-kibana-running.png)

---

### 5) Deploy Fluentd (DaemonSet) to collect cluster logs

```bash
kubectl apply -n logging -f fluentd-configmap.yaml
kubectl apply -n logging -f fluentd-daemonset.yaml
kubectl get ds -n logging
kubectl get pods -n logging -l app=fluentd -o wide
```

📸 `screenshots/03-fluentd-daemonset.png`
![Fluentd DaemonSet](screenshots/03-fluentd-daemonset.png)

Confirm Fluentd is actually shipping logs:

```bash
kubectl logs -n logging ds/fluentd --tail=80
```

---

### 6) Generate demo logs (simulate an incident)

Deploy a noisy demo app that outputs errors:

```bash
kubectl create ns demo
kubectl apply -n demo -f demo-app.yaml
kubectl get pods -n demo -w
```

Confirm the demo app is producing logs:

```bash
kubectl logs -n demo deploy/demo-logger --tail=50
```

📸 `screenshots/04-demo-logs.png`
![Demo Logs](screenshots/04-demo-logs.png)

---

### 7) Access Kibana UI (port-forward)

```bash
kubectl port-forward -n logging svc/kibana 5601:5601
```

Open in browser:

* [http://localhost:5601](http://localhost:5601)

📸 `screenshots/05-kibana-home.png` 
Should show: Kibana UI is reachable.

---

### 8) Create a Data View (Index Pattern)

In Kibana:

* **Stack Management → Data Views**
* Create data view: `logstash-*` *(match Fluentd index name)*
* Time field: `@timestamp`

📸 `screenshots/06-index-pattern.png`
![Index Pattern](screenshots/06-index-pattern.png)

---

### 9) Search logs like an incident responder

Example queries in Kibana:

* `error`
* `kubernetes.namespace_name:"demo" AND "timeout"`
* `kubernetes.pod_name:*demo* AND "500"`

📸 `screenshots/07-kibana-error-search.png`
![Error Search](screenshots/07-kibana-error-search.png)

📸 `screenshots/08-filter-by-namespace.png`
![Filter by Namespace](screenshots/08-filter-by-namespace.png)

---

## Outcome

After deploying EFK on Minikube, I can:

* collect logs from **every pod** into one place
* search by:

  * keyword (`error`, `timeout`, `500`)
  * time window
  * namespace/pod/container
* correlate failures across services during incidents
* reduce troubleshooting time because log search is instant

---

## Troubleshooting

### Kibana loads but shows “No data”

Check Fluentd + Elasticsearch logs:

```bash
kubectl logs -n logging ds/fluentd --tail=120
kubectl logs -n logging deploy/elasticsearch --tail=120
```

Confirm indices exist:

```bash
kubectl port-forward -n logging svc/elasticsearch 9200:9200
curl -s http://localhost:9200/_cat/indices?v
```

---

### Elasticsearch keeps restarting (OOM / not enough resources)

Restart Minikube with more memory:

```bash
minikube stop
minikube start --cpus=4 --memory=8192
```

---

### Fluentd is running but nothing appears in Elasticsearch

Verify ConfigMap and Fluentd output settings:

```bash
kubectl get cm -n logging fluentd-config -o yaml | sed -n '1,220p'
kubectl logs -n logging ds/fluentd --tail=200
```

---

### Kibana can’t connect to Elasticsearch

Check Kibana env vars and service DNS:

```bash
kubectl describe pod -n logging -l app=kibana | sed -n '1,220p'
kubectl get svc -n logging
```

---

## Cleanup

```bash
kubectl delete ns demo --ignore-not-found
kubectl delete ns logging --ignore-not-found
minikube stop
```
