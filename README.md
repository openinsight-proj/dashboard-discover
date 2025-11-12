# Dashboard-discover

## Build

```dockerfile
docker build -t ghcr.io/openinsight-proj/dashboard-discover:dev .
```

## Run

### binary
```yaml
./dashboard-discover \
  --log-level=debug \
  --kubeconfig=./kubeconfig.yaml \
  --watch-resource-labels=dashboard=true \
  --folder=./tmp \
  --namespace-to-folder=true
```

### docker
```shell
docker run --rm -it \
  -p 8080:8080 \
  -v ./kubeconfig.yaml:/app/kubeconfig.yaml \
  ghcr.io/openinsight-proj/dashboard-discover:dev \
  --kubeconfig /app/kubeconfig.yaml
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

2. deploy
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
      "panels": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PBFA97CFB590B2093"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "thresholds"
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": 0
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 4,
            "w": 3,
            "x": 0,
            "y": 0
          },
          "id": 1,
          "options": {
            "colorMode": "value",
            "graphMode": "area",
            "justifyMode": "auto",
            "orientation": "auto",
            "percentChangeColorMode": "standard",
            "reduceOptions": {
              "calcs": [
                "lastNotNull"
              ],
              "fields": "",
              "values": false
            },
            "showPercentChange": false,
            "textMode": "auto",
            "wideLayout": true
          },
          "pluginVersion": "12.1.3",
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PBFA97CFB590B2093"
              },
              "editorMode": "code",
              "expr": "count(up)",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "New panel",
          "type": "stat"
        }
      ],
      "preload": false,
      "schemaVersion": 41,
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-6h",
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
      "panels": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PBFA97CFB590B2093"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "thresholds"
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": 0
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 4,
            "w": 3,
            "x": 0,
            "y": 0
          },
          "id": 1,
          "options": {
            "colorMode": "value",
            "graphMode": "area",
            "justifyMode": "auto",
            "orientation": "auto",
            "percentChangeColorMode": "standard",
            "reduceOptions": {
              "calcs": [
                "lastNotNull"
              ],
              "fields": "",
              "values": false
            },
            "showPercentChange": false,
            "textMode": "auto",
            "wideLayout": true
          },
          "pluginVersion": "12.1.3",
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PBFA97CFB590B2093"
              },
              "editorMode": "code",
              "expr": "count(up)",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "New panel",
          "type": "stat"
        }
      ],
      "preload": false,
      "schemaVersion": 41,
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-6h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "browser",
      "title": "New dashboard from CR(v4)",
      "uid": "f388a13d-aeb8-495c-b2da-2ad09e996c0c",
      "version": 0
    }
```