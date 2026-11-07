# Cấu trúc hiện tại của Chatbox

```text
Máy chủ AWS EC2
├─ Caddy (HTTPS)
│  ├─ chatbox.cyou → frontend
│  ├─ auth.chatbox.cyou → Keycloak
│  └─ grafana.chatbox.cyou → Grafana
│
└─ K3s
   ├─ namespace chatbox1
   │  ├─ frontend (React + Nginx)
   │  ├─ backend (Go API)
   │  ├─ MongoDB (dữ liệu Chatbox)
   │  └─ 9Router (cổng gọi model AI)
   │
   ├─ namespace keycloak
   │  ├─ Keycloak (đăng nhập, token và role)
   │  └─ PostgreSQL (dữ liệu Keycloak)
   │
   ├─ namespace monitoring
   │  ├─ Grafana (dashboard)
   │  ├─ Prometheus (metric)
   │  ├─ Alertmanager (cảnh báo)
   │  ├─ kube-state-metrics (trạng thái tài nguyên K3s)
   │  ├─ node-exporter (metric máy chủ)
   │  ├─ Loki (log)
   │  └─ Alloy (thu log và Kubernetes Event)
   │
   └─ namespace argocd
      ├─ Argo CD Application: Chatbox
      └─ Argo CD Application: monitoring
```
