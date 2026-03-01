# Centralized Logging on Minikube (EFK) — Incident Log Search

## Context

In a real incident, I don’t want to jump pod-to-pod running `kubectl logs`. I want **one place** to search logs fast, filter by **namespace/pod**, and confirm what happened.

This project is a lightweight **centralized logging stack (EFK = Elasticsearch + Fluentd + Kibana)** running on **Minikube**.

---

## Problem

During incidents, logs are **scattered**:

* Logs live inside pods (pods restart, logs rotate)
* Many namespaces/services make it hard to know where to start
* `kubectl logs` is useful, but incident response needs:

  * full-text **search**
  * **time-based** filtering
  * correlation across services
  * quick pattern detection (`error`, `timeout`, `500`)

**Bottom line:** I need one place to store + search logs across the cluster.

---

## Solution

I deployed an **EFK pipeline** on Minikube:

* **Fluentd (DaemonSet)** collects node/pod logs across the cluster
* Fluentd forwards logs to **Elasticsearch** (storage + indexing)
* **Kibana** reads from Elasticsearch so I can search + filter logs during incidents

---

## Architecture

![EFK Architecture](screenshots/architecture.png)

Logs from all namespaces → Fluentd on each node → Elasticsearch (index/store) → Kibana (search/filter).

---

## What I did  + screenshots

### 1) Start Minikube with enough resources

* Started Minikube (Elasticsearch needs memory/CPU)
* Verified the node is Ready

📸 `screenshots/00-minikube-node-ready.png` — Should show: node `Ready`

---

### 2) Create the logging namespace

* Created a dedicated namespace to keep logging components separated

---

### 3) Deploy Elasticsearch (single-node for demo)

* Applied the Elasticsearch manifest
* Verified pods are running + service/endpoints exist

📸 `screenshots/01-efk-pods-running.png` — Should show: EFK pods running in `logging`
📸 `screenshots/02-elasticsearch-service.png` — Should show: Elasticsearch service + endpoints

---

### 4) Deploy Kibana

* Applied the Kibana manifest
* Verified Kibana pod and service are running

📸 `screenshots/05-kibana-running.png` — Should show: Kibana pod Running + service available
📸 `screenshots/05-kibana-home.png` — Should show: Kibana UI reachable in browser

---

### 5) Deploy Fluentd (DaemonSet) to collect logs

* Applied Fluentd ConfigMap + DaemonSet
* Verified Fluentd is running on the node(s)

📸 `screenshots/03-fluentd-daemonset.png` — Should show: Fluentd DaemonSet ready + pods on node(s)

---

### 6) Generate demo logs (simulate an incident)

* Deployed a demo app that produces noisy/error logs
* Verified logs are being produced

📸 `screenshots/04-demo-logs.png` — Should show: demo app logs with errors/noise

---

### 7) In Kibana: create a Data View (Index Pattern)

* Created a Data View so Kibana can query the indices

📸 `screenshots/06-index-pattern.png` — Should show: Data View created successfully

---

### 8) Search + filter like an incident responder

* Searched for errors
* Filtered by namespace/pod to isolate the issue fast

📸 `screenshots/07-kibana-error-search.png` — Should show: searching `error` results
📸 `screenshots/08-filter-by-namespace.png` — Should show: filter by `demo` namespace / pod

---

## Result

After this setup, I can:

* collect logs from **every pod** into one place
* search by **keyword** (`error`, `timeout`, `500`)
* filter by **time window + namespace + pod/container**
* troubleshoot faster because log search is instant (no more pod-by-pod guessing)

---

## Troubleshooting

### Kibana loads but shows “No data”

* Fluentd may not be shipping logs, or indices aren’t created yet
* Check Fluentd + Elasticsearch logs, and confirm indices exist

### Elasticsearch keeps restarting (OOM)

* Minikube doesn’t have enough memory/CPU
* Restart Minikube with more resources

### Fluentd is running but nothing appears in Elasticsearch

* Check Fluentd ConfigMap output settings and Fluentd logs

### Kibana can’t connect to Elasticsearch

* Verify Kibana env vars and service DNS inside the `logging` namespace

---

## Useful CLI 

### Start cluster

```bash
minikube start --cpus=4 --memory=8192
kubectl get nodes -o wide
```

### Deploy EFK

```bash
kubectl create namespace logging

kubectl apply -n logging -f elasticsearch.yaml
kubectl apply -n logging -f kibana.yaml
kubectl apply -n logging -f fluentd-configmap.yaml
kubectl apply -n logging -f fluentd-daemonset.yaml

kubectl get pods -n logging -o wide
kubectl get svc -n logging
kubectl get ds -n logging
```

### Deploy demo app + confirm logs

```bash
kubectl create ns demo
kubectl apply -n demo -f demo-app.yaml
kubectl get pods -n demo -w
kubectl logs -n demo deploy/demo-logger --tail=50
```

### Access Kibana

```bash
kubectl port-forward -n logging svc/kibana 5601:5601
```

### Quick verify indices in Elasticsearch

```bash
kubectl port-forward -n logging svc/elasticsearch 9200:9200
curl -s http://localhost:9200/_cat/indices?v
```

### Useful log checks

```bash
kubectl logs -n logging ds/fluentd --tail=120
kubectl logs -n logging deploy/elasticsearch --tail=120
```

---

## Cleanup

```bash
kubectl delete ns demo --ignore-not-found
kubectl delete ns logging --ignore-not-found
minikube stop
```
