# Dashboard-discover

This tool watches Kubernetes resources (such as ConfigMap or custom resources like GrafanaDashboards.integreatly.org)
based on specified labels, and automatically saves their content to local files in a configurable directory structure.


## Build

```binary
go build -o dashboard-discover .
```

```dockerfile
docker build -t ghcr.io/openinsight-proj/dashboard-discover:dev .
```

## Usage

### Flags

```shell
Flags:
      --configmap-selector string           Kubernetes label selector to filter which ConfigMaps to watch (format: key=value, supports multiple via comma separation) (default "dashboard=true,grafana=true")
      --enable-reload                       enable reload
      --folder string                       folder to save file (default "./tmp")
      --grafanadashboard-selector string    Kubernetes label selector to filter which grafanadashboards.integreatly.org to watch (format: key=value, supports multiple via comma separation) (default "dashboard=true")
  -h, --help                                help for dashboard-discover
      --kubeconfig string                   path to the kubeconfig file, if not specified, default is $HOME/.kube/config or from env var KUBECONFIG
      --label-as-subfolder string           save files into a subfolder named by the value of a specific resource label. Takes priority over the --namespace-as-subfolder flag if set (default "folder")
      --log-level string                    log level (default "info")
      --namespace-as-subfolder              Save files into subfolders named after their namespace (default true)
      --request-password REQUEST_PASSWORD   password to whom the request is sent, or from REQUEST_PASSWORD env var
      --request-reload-url string           URL to which send a request after a configmap got reloaded (default "http://localhost:3000/api/admin/provisioning/dashboards/reload")
      --request-user REQUEST_USER           user to whom the request is sent, or from REQUEST_USER env var
      --telemetry-address string            health server address (default ":8082")

```

### Binary
```yaml
./dashboard-discover \
  --log-level=debug \
  --kubeconfig=./kubeconfig.yaml \
  --folder=./tmp
```

### Docker
```shell
docker run --rm -it \
  -p 8080:8080 \
  -v ./kubeconfig.yaml:/app/kubeconfig.yaml \
  ghcr.io/openinsight-proj/dashboard-discover:dev \
  --log-level=debug \
  --kubeconfig /app/kubeconfig.yaml \
  --folder=./tmp
```

### Kube

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
      serviceAccountName: grafana-serviceaccount
      volumes:
        - name: grafana-plugins
          emptyDir: {}
      containers:
        - name: dashboard-discover
          image: "ghcr.io/openinsight-proj/dashboard-discover:dev"
          imagePullPolicy: IfNotPresent
          args:
          - "--folder=/opt/plugins/dashboards"
          volumeMounts:
            - name: grafana-plugins
              mountPath: /opt/plugins
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
    grafana: 'true'
    folder: "test"
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