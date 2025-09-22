# 📊 Prometheus Architecture Explained

This document provides a detailed explanation of the Prometheus monitoring architecture based on the provided diagram.
![prometheus-architecture.pnd](./img/prometheus-architecture.pnd)

---

## 🧱 Core Components

### 1. **Prometheus Server**

The central component that:

- **Retrieves** metrics from targets.
- Stores metrics in **TSDB** (Time Series Database).
- Exposes a web interface and API via an **HTTP Server**.

Prometheus operates on a **pull-based model**, regularly scraping metrics from configured targets.

---

## 🎯 Prometheus Targets

**Targets** are endpoints that expose metrics at `/metrics`, typically using:

- **Exporters** like Node Exporter, cAdvisor, etc.
- Custom applications exposing Prometheus metrics.

**Pushgateway** is used for targets that cannot be scraped directly (e.g., short-lived jobs).

---

## 🚀 Pushgateway

Used for **short-lived jobs** that push metrics to Prometheus via Pushgateway before terminating.

Prometheus then **pulls** metrics from Pushgateway as part of its regular scraping.

---

## 🔍 Service Discovery

Prometheus can discover targets dynamically through:

- **Kubernetes**: Detects pods, services, and endpoints.
- **file_sd**: Static file-based service discovery (YAML/JSON).

---

## 💾 Storage (TSDB)

Prometheus stores data in a **time-series database** on local **HDD/SSD**.
Each metric has:

- A **timestamp**
- **Labels** (key-value pairs)
- A **value**

---

## 📤 Alerting: Alertmanager

Prometheus pushes alerts to **Alertmanager**, which:

- Deduplicates and groups alerts.
- Sends notifications to external systems like:
  - 📞 **PagerDuty**
  - 📧 **Email**
  - 🔔 **Other platforms**

---

## 📈 Data Visualization and Export

### a. **Prometheus Web UI**

- Built-in web UI for quick queries and exploration.

### b. **Grafana**

- Advanced dashboard and visualization tool.
- Integrates with Prometheus as a data source.

### c. **API Clients**

- Applications or scripts that query metrics via Prometheus HTTP API.

---

## 🧠 PromQL (Prometheus Query Language)

Prometheus uses **PromQL** to:

- Query and aggregate metrics.
- Create dashboards and alerts.

**Example:**

```promql
rate(http_requests_total[5m])
```

---

## 🔄 Data Flow Summary

1. Exporters expose metrics.
2. Prometheus scrapes metrics.
3. TSDB stores the data.
4. PromQL is used for querying.
5. Alerts are pushed to Alertmanager.
6. Grafana and Web UI visualize the data.

---

## 🧪 Real-World Use Case: Kubernetes

- Prometheus uses **Kubernetes service discovery** to find pods.
- Scrapes metrics from **node-exporter** and **kube-state-metrics**.
- Stores data in TSDB.
- Sends alerts for pod failures.
- Grafana displays cluster health dashboards.

---

> ✅ This architecture makes Prometheus a powerful and flexible system for monitoring cloud-native and containerized applications.
