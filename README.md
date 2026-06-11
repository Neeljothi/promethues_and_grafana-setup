# Prometheus & Grafana on Kubernetes

A complete guide to deploying Prometheus and Grafana on a Kubernetes cluster using Helm for monitoring and observability.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Installation](#installation)
  - [1. Add Helm Repositories](#1-add-helm-repositories)
  - [2. Create a Namespace](#2-create-a-namespace)
  - [3. Install Prometheus](#3-install-prometheus)
  - [4. Install Grafana](#4-install-grafana)
- [Accessing the Dashboards](#accessing-the-dashboards)
  - [Prometheus UI](#prometheus-ui)
  - [Grafana UI](#grafana-ui)
- [Connecting Grafana to Prometheus](#connecting-grafana-to-prometheus)
- [Configuration](#configuration)
  - [Custom Prometheus Scrape Config](#custom-prometheus-scrape-config)
  - [Persistent Storage](#persistent-storage)
  - [Grafana Admin Password](#grafana-admin-password)
- [Importing Grafana Dashboards](#importing-grafana-dashboards)
- [Upgrading](#upgrading)
- [Uninstalling](#uninstalling)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

This guide covers deploying a production-ready monitoring stack on Kubernetes using:

- **[Prometheus](https://prometheus.io/)** — an open-source metrics collection and alerting system
- **[Grafana](https://grafana.com/)** — an open-source visualization and dashboarding platform

Both are deployed via the **[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)** Helm chart, which bundles Prometheus, Alertmanager, Grafana, and a set of pre-built Kubernetes dashboards.

---

## Prerequisites

Ensure the following tools are installed and configured before proceeding:

| Tool | Minimum Version | Purpose |
|------|----------------|---------|
| `kubectl` | v1.24+ | Kubernetes CLI |
| `helm` | v3.10+ | Kubernetes package manager |
| Kubernetes cluster | v1.24+ | Local (minikube, kind) or cloud (EKS, GKE, AKS) |

Verify your setup:

```bash
kubectl version --client
helm version
kubectl cluster-info
```

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Kubernetes Cluster              │
│                                                 │
│  ┌─────────────────┐    ┌───────────────────┐   │
│  │   Prometheus    │◄───│  Node Exporters   │   │
│  │   (metrics DB)  │    │  (per-node agent) │   │
│  └────────┬────────┘    └───────────────────┘   │
│           │                                     │
│           │  scrapes metrics                    │
│           ▼                                     │
│  ┌─────────────────┐    ┌───────────────────┐   │
│  │    Grafana      │    │  Alertmanager     │   │
│  │  (dashboards)   │    │  (alert routing)  │   │
│  └─────────────────┘    └───────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

## Installation

### 1. Add Helm Repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Confirm the repos were added:

```bash
helm repo list
```

---

### 2. Create a Namespace

It is best practice to isolate the monitoring stack in its own namespace:

```bash
kubectl create namespace monitoring
```

---

### 3. Install Prometheus

#### Option A — Install via kube-prometheus-stack (Recommended)

This installs Prometheus, Alertmanager, Grafana, and pre-built Kubernetes dashboards in a single chart:

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

To install with a custom values file:

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values custom-values.yaml
```

#### Option B — Install Prometheus Standalone

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace
```

Verify the pods are running:

```bash
kubectl get pods -n monitoring
```

Expected output:

```
NAME                                                  READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0     2/2     Running   0          2m
kube-prometheus-stack-grafana-xxxxxxxxxx-xxxxx        3/3     Running   0          2m
kube-prometheus-stack-operator-xxxxxxxxxx-xxxxx       1/1     Running   0          2m
prometheus-kube-prometheus-stack-prometheus-0         2/2     Running   0          2m
```

---

### 4. Install Grafana

> **Note:** If you installed `kube-prometheus-stack` in Step 3, Grafana is already included. Skip to [Accessing the Dashboards](#accessing-the-dashboards).

To install Grafana standalone:

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword='your-secure-password' \
  --set service.type=ClusterIP
```

Verify the deployment:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
```

---

## Accessing the Dashboards

### Prometheus UI

Forward the Prometheus service to your local machine:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open your browser at: [http://localhost:9090](http://localhost:9090)

---

### Grafana UI

#### Retrieve the admin password (kube-prometheus-stack)

```bash
kubectl get secret -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

#### For standalone Grafana installation

```bash
kubectl get secret -n monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

Forward the Grafana service:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open your browser at: [http://localhost:3000](http://localhost:3000)

Default credentials:
- **Username:** `admin`
- **Password:** (from the secret above)

---

## Connecting Grafana to Prometheus

When using `kube-prometheus-stack`, Grafana is pre-configured with Prometheus as its default data source.

For a **standalone** setup, add the data source manually:

1. Log in to Grafana
2. Go to **Connections → Data Sources → Add data source**
3. Select **Prometheus**
4. Set the URL to the internal service address:

```
http://prometheus-server.monitoring.svc.cluster.local:80
```

5. Click **Save & Test** — you should see a green success banner

---

## Configuration

### Custom Prometheus Scrape Config

Create a `custom-values.yaml` file to add your own scrape targets:

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: my-app
        static_configs:
          - targets:
              - my-app-service.default.svc.cluster.local:8080
```

Apply the configuration:

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values custom-values.yaml
```

---

### Persistent Storage

By default, data is stored in ephemeral volumes and will be lost on pod restarts. To enable persistent storage:

```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

grafana:
  persistence:
    enabled: true
    size: 10Gi
```

---

### Grafana Admin Password

Set a secure admin password during installation:

```yaml
grafana:
  adminPassword: "your-secure-password"
```

Or store it as a Kubernetes Secret and reference it:

```bash
kubectl create secret generic grafana-admin-secret \
  --from-literal=admin-password='your-secure-password' \
  -n monitoring
```

```yaml
grafana:
  admin:
    existingSecret: grafana-admin-secret
    passwordKey: admin-password
```

---

## Importing Grafana Dashboards

`kube-prometheus-stack` ships with pre-built dashboards for Kubernetes cluster monitoring. You can also import community dashboards from [grafana.com/dashboards](https://grafana.com/grafana/dashboards/).

**Popular dashboard IDs:**

| Dashboard | ID |
|-----------|----|
| Kubernetes Cluster Overview | `315` |
| Node Exporter Full | `1860` |
| Kubernetes Pods | `6417` |
| Kubernetes Deployments | `8588` |

**To import a dashboard:**

1. In Grafana, go to **Dashboards → Import**
2. Enter the dashboard ID (e.g., `1860`)
3. Click **Load**
4. Select your Prometheus data source
5. Click **Import**

---

## Upgrading

To upgrade to the latest chart version:

```bash
helm repo update

helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values custom-values.yaml
```

Check the current release status:

```bash
helm status kube-prometheus-stack -n monitoring
helm history kube-prometheus-stack -n monitoring
```

---

## Uninstalling

```bash
helm uninstall kube-prometheus-stack -n monitoring
```

Clean up remaining CustomResourceDefinitions (CRDs):

```bash
kubectl delete crd \
  alertmanagerconfigs.monitoring.coreos.com \
  alertmanagers.monitoring.coreos.com \
  podmonitors.monitoring.coreos.com \
  probes.monitoring.coreos.com \
  prometheuses.monitoring.coreos.com \
  prometheusrules.monitoring.coreos.com \
  servicemonitors.monitoring.coreos.com \
  thanosrulers.monitoring.coreos.com
```

Delete the namespace:

```bash
kubectl delete namespace monitoring
```

---

## Troubleshooting

**Pods stuck in `Pending` state**

```bash
kubectl describe pod <pod-name> -n monitoring
```

This is often caused by insufficient cluster resources or missing PersistentVolumes. Check node resource availability with:

```bash
kubectl top nodes
```

---

**Cannot access Prometheus or Grafana via port-forward**

Verify the correct service name:

```bash
kubectl get svc -n monitoring
```

Then re-run port-forward using the exact service name from the output.

---

**Grafana shows "No data" in dashboards**

- Confirm the Prometheus data source is configured and shows **Data source connected**
- Check that Prometheus is successfully scraping targets at `http://localhost:9090/targets`
- Ensure the time range in Grafana is set to the last 15 minutes or more

---

**Check Prometheus logs**

```bash
kubectl logs -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -c prometheus
```

**Check Grafana logs**

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana
```

---

## References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Community Helm Charts](https://github.com/prometheus-community/helm-charts)
- [Grafana Helm Chart](https://github.com/grafana/helm-charts)
- [Grafana Dashboard Library](https://grafana.com/grafana/dashboards/)
