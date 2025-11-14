# Dashboard-discover

This tool watches Kubernetes resources (such as ConfigMap or custom resources like GrafanaDashboards.integreatly.org)
based on specified labels, and automatically saves their content to local files in a configurable directory structure. 
It is designed to support Grafana dashboard provisioning by syncing dashboards from the cluster to the filesystem, 
with optional reload notifications sent to Grafana’s HTTP API.

## Flags

| Flag                      | Default                                                          | Description                                                                                                                             |
|---------------------------|------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| `--log-level`             | `info`                                                           | Log level (e.g., `debug`, `info`, `warn`, `error`).                                                                                     |
| `--health-server-addr`    | `:8082`                                                          | Address for the health check HTTP server.                                                                                               |
| `--kubeconfig`            | *unset*                                                          | Path to the kubeconfig file. If not specified, defaults to `$HOME/.kube/config` or the `KUBECONFIG` environment variable.               |
| `--watch-resource-labels` | `dashboard=true`                                                 | Kubernetes label selector used to identify resources to watch (e.g., `app=grafana`).                                                    |
| `--sub-folder-label`      | *empty*                                                          | If set, creates subfolders using the value of this label from each watched resource. Takes precedence over `--namespace-to-folder`.     |
| `--folder`                | `./tmp`                                                          | Local directory where synced resources are saved.                                                                                       |
| `--namespace-to-folder`   | `true`                                                           | Organize files into subdirectories by namespace (ignored if `--sub-folder-label` is set).                                               |
| `--resources`             | `["configmap", "grafanadashboards.integreatly.org"]`             | Resource types to watch. Can be specified multiple times (e.g., `--resources=configmap --resources=grafanadashboards.integreatly.org`). |
| `--enable-reload`         | `false`                                                          | Enable sending an HTTP POST request to reload Grafana dashboards after updates.                                                         |
| `--request-reload-url`    | `http://localhost:3000/api/admin/provisioning/dashboards/reload` | URL to trigger Grafana dashboard provisioning reload.                                                                                   |
| `--request-user`          | *(from `REQUEST_USER` env var)*                                  | Username for basic authentication when sending reload requests.                                                                         |
| `--request-password`      | *(from `REQUEST_PASSWORD` env var)*                              | Password for basic authentication when sending reload requests.                                                                         |

> Note: If --request-user or --request-password are not provided via flags, the tool will attempt to read them from the 
> REQUEST_USER and REQUEST_PASSWORD environment variables respectively.

## Build

```binary
go build -o dashboard-discover .
```

```dockerfile
docker build -t ghcr.io/openinsight-proj/dashboard-discover:dev .
```

## Usage

### binary
```yaml
./dashboard-discover \
  --log-level=debug \
  --kubeconfig=./kubeconfig.yaml \
  --watch-resource-labels=dashboard=true \
  --folder=./tmp
```

### docker
```shell
docker run --rm -it \
  -p 8080:8080 \
  -v ./kubeconfig.yaml:/app/kubeconfig.yaml \
  ghcr.io/openinsight-proj/dashboard-discover:dev \
  --log-level=debug \
  --kubeconfig /app/kubeconfig.yaml \
  --watch-resource-labels=dashboard=true \
  --folder=./tmp
```

### kube

1. create RBAC
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana-serviceaccount
  namespace: insight-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dashboard-discover
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["integreatly.org"]
    resources: ["grafanadashboards"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-discover
subjects:
  - kind: ServiceAccount
    name: grafana-serviceaccount
    namespace: insight-system
roleRef:
  kind: ClusterRole
  name: dashboard-discover
  apiGroup: rbac.authorization.k8s.io
```

2. deploy Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-discover
  namespace: default
spec:
  template:
    spec:
      containers:
        - name: dashboard-discover
          image: "ghcr.io/openinsight-proj/dashboard-discover:dev"
          imagePullPolicy: IfNotPresent
          args:
          - "--log-level=debug"
          - "--folder=/opt/plugins/dashboards"
          - "--watch-resource-labels=dashboard=true"
          - "--namespace-to-folder=true"
          - "--health-server-addr=:8082"
          volumeMounts:
            - name: grafana-plugins
              mountPath: /opt/plugins
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8082
              scheme: HTTP
            initialDelaySeconds: 120
            timeoutSeconds: 5
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 6
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8082
              scheme: HTTP
            initialDelaySeconds: 30
            timeoutSeconds: 5
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 6
```

3. test
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-configmap
  namespace: default
  labels:
    dashboard: 'true'
data:
  hello.json: |-
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "grafana",
              "uid": "-- Grafana --"
            },
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 0,
      "links": [],
      "panels": [],
      "preload": false,
      "schemaVersion": 41,
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-1h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "browser",
      "title": "New dashboard from Configmap",
      "uid": "f388a13d-aeb8-495c-b2da-2ad09e996c0d",
      "version": 0
    }
---
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDashboard
metadata:
  labels:
    dashboard: 'true'
  name: simple-dashboard
  namespace: default
spec:
  json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "grafana",
              "uid": "-- Grafana --"
            },
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 0,
      "links": [],
      "panels": [],
      "preload": false,
      "schemaVersion": 41,
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-1h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "browser",
      "title": "New dashboard from CR(v4)",
      "uid": "f388a13d-aeb8-495c-b2da-2ad09e996c0c",
      "version": 0
    }
```