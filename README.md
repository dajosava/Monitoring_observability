# 🚀 Despliegue de Stack de Observabilidad (Prometheus + Grafana) en Minikube

**Fecha de documentación:** 2025-10-21

---

## 🧩 Requisitos previos

Asegúrate de tener los siguientes componentes instalados y funcionando en tu PC:

- ✅ **Virtualización habilitada** (BIOS → Intel VT-x o AMD-V → *Enabled*)
- ✅ **Windows 11 + WSL2 + Ubuntu**
- ✅ **Docker Desktop o Minikube** (recomendado Minikube)
- ✅ **kubectl y Helm 3**

Verifica tu entorno:

```bash
wsl -l -v
kubectl version --short
helm version
minikube version
```

---

## ⚙️ 1. Iniciar Minikube

Ejecuta:

```bash
minikube start --driver=docker --cpus=4 --memory=8g --disk-size=20g
kubectl get nodes
kubectl cluster-info
```

---

## 📦 2. Script de Despliegue: Prometheus + Grafana

Guarda el siguiente script como `deploy-monitoring.sh`:

```bash
#!/usr/bin/env bash
# deploy-monitoring.sh
# Despliega kube-prometheus-stack (Prometheus + Grafana) en el namespace "monitoring"

set -euo pipefail

NAMESPACE="monitoring"
RELEASE="prometheus"
CHART="prometheus-community/kube-prometheus-stack"
CHART_VERSION=""

msg()  { printf "\n\033[1;36m[INFO]\033[0m %s\n" "$*"; }
err()  { printf "\n\033[1;31m[ERROR]\033[0m %s\n" "$*" >&2; }
need() { command -v "$1" >/dev/null 2>&1 || { err "falta '$1' en PATH"; exit 1; }; }

need kubectl
need helm

msg "Verificando acceso al clúster…"
kubectl version --short >/dev/null
kubectl get nodes >/dev/null

if ! kubectl get ns "${NAMESPACE}" >/dev/null 2>&1; then
  msg "Creando namespace '${NAMESPACE}'…"
  kubectl create namespace "${NAMESPACE}"
else
  msg "Namespace '${NAMESPACE}' ya existe"
fi

if ! helm repo list | grep -q prometheus-community; then
  msg "Agregando repo prometheus-community…"
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
fi
msg "Actualizando índices de charts…"
helm repo update

if helm -n "${NAMESPACE}" status "${RELEASE}" >/dev/null 2>&1; then
  msg "Realizando upgrade de ${RELEASE}…"
  helm upgrade "${RELEASE}" "${CHART}" -n "${NAMESPACE}" ${CHART_VERSION}
else
  msg "Instalando ${RELEASE}…"
  helm install "${RELEASE}" "${CHART}" -n "${NAMESPACE}" --create-namespace ${CHART_VERSION}
fi

msg "Esperando pods en estado Running…"
kubectl -n "${NAMESPACE}" wait --for=condition=Ready pods --all --timeout=10m || true
kubectl -n "${NAMESPACE}" get pods -o wide

GRAFANA_USER="admin"
GRAFANA_PASS="prom-operator"

msg "Servicios desplegados:"
kubectl -n "${NAMESPACE}" get svc

cat <<EOF

=============================================================
✅ Despliegue completado

Para acceder a Grafana localmente:
  kubectl -n ${NAMESPACE} port-forward svc/${RELEASE}-grafana 3000:80

Abrir en navegador:
  👉 http://localhost:3000

Credenciales:
  user: ${GRAFANA_USER}
  pass: ${GRAFANA_PASS}

Para acceder a Prometheus:
  kubectl -n ${NAMESPACE} port-forward svc/${RELEASE}-kube-prometheus-prometheus 9090:9090
  👉 http://localhost:9090

Para desinstalar:
  helm uninstall ${RELEASE} -n ${NAMESPACE} && kubectl delete ns ${NAMESPACE}
=============================================================
EOF
```

---

## 🚀 3. Ejecución

Dale permisos y ejecútalo:

```bash
chmod +x deploy-monitoring.sh
./deploy-monitoring.sh
```

---

## 🌐 4. Acceso a Interfaces

| Servicio | URL Local | Usuario | Contraseña |
|-----------|------------|----------|-------------|
| Grafana | [http://localhost:3000](http://localhost:3000) | `admin` | `prom-operator` |
| Prometheus | [http://localhost:9090](http://localhost:9090) | — | — |

---

## 🧹 5. Limpieza (opcional)

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
minikube stop
```

---

## 🧠 Notas finales

- El chart `kube-prometheus-stack` instala:
  - Prometheus Server
  - Alertmanager
  - Node Exporter
  - Grafana con dashboards preinstalados
- Ideal para practicar **observabilidad**, **alertas**, **dashboards personalizados** y **exporters** en entrevistas técnicas.

---

Autor: **David Salazar (Manakin Labs)**  
Repositorio: [https://github.com/ManakinLabs](https://github.com/ManakinLabs)
