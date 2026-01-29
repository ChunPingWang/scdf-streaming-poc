# Spring Cloud Dataflow Streaming PoC

在 Kind (Kubernetes in Docker) 上運行 Spring Cloud Dataflow 串流處理環境的 PoC 專案。

## 架構概覽

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kind Kubernetes Cluster                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   MariaDB   │  │    Kafka    │  │  Spring Cloud Dataflow  │  │
│  │   (scdf)    │  │   (kafka)   │  │   Server + Skipper      │  │
│  │  Port:3306  │  │  Port:9092  │  │  Dashboard:30080        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              NGINX Ingress Controller                        │ │
│  │              HTTP:80 / HTTPS:443                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## 環境需求

- Docker Desktop 4.x+ (至少配置 8GB RAM, 4 CPU)
- Kind v0.20+
- kubectl v1.28+
- Helm v3.12+

## 快速開始

### 1. 建立 Kind 叢集

```bash
kind create cluster --config kind-config.yaml
```

### 2. 建立 Namespaces

```bash
kubectl create namespace scdf
kubectl create namespace kafka
```

### 3. 部署 Kafka

```bash
kubectl apply -f kafka-cluster.yaml
```

等待 Kafka 就緒：
```bash
kubectl wait --for=condition=ready pod -l app=kafka -n kafka --timeout=180s
```

### 4. 部署 MariaDB

```bash
kubectl apply -f mysql.yaml
```

等待 MariaDB 就緒：
```bash
kubectl wait --for=condition=ready pod -l app=mysql -n scdf --timeout=120s
```

### 5. 部署 Spring Cloud Dataflow

```bash
kubectl apply -f scdf.yaml
```

等待 SCDF 就緒（約需 5 分鐘）：
```bash
kubectl wait --for=condition=ready pod -l app=skipper -n scdf --timeout=300s
kubectl wait --for=condition=ready pod -l app=dataflow-server -n scdf --timeout=300s
```

### 6. 安裝 Ingress Controller（選用）

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl apply -f ingress.yaml
```

## 存取服務

### SCDF Dashboard

透過 NodePort 存取：
- URL: http://localhost:30080/dashboard

透過 Ingress 存取（需安裝 Ingress Controller）：
- URL: http://localhost/dashboard

### Skipper

- URL: http://localhost:30081

### Kafka

- 叢集內部: `kafka.kafka.svc.cluster.local:9092`
- 外部存取: `localhost:30092`

## 檔案說明

| 檔案 | 說明 |
|------|------|
| `kind-config.yaml` | Kind 叢集配置，包含 port mapping |
| `kafka-cluster.yaml` | Apache Kafka 部署配置（KRaft 模式） |
| `mysql.yaml` | MariaDB 部署配置 |
| `scdf.yaml` | Spring Cloud Dataflow Server 和 Skipper 配置 |
| `scdf-values.yaml` | SCDF Helm values（參考用） |
| `ingress.yaml` | NGINX Ingress 配置 |

## 驗證部署

檢查所有 Pods 狀態：
```bash
kubectl get pods -n scdf
kubectl get pods -n kafka
```

預期輸出：
```
NAME                               READY   STATUS    RESTARTS   AGE
dataflow-server-xxx                1/1     Running   0          5m
mysql-xxx                          1/1     Running   0          10m
skipper-xxx                        1/1     Running   0          5m
```

## 建立 Stream 範例

### 簡單的 Log Stream

在 SCDF Dashboard 中建立 Stream：

```
http --port=8080 | log
```

或使用 REST API：
```bash
# 取得 Server Pod
SCDF_POD=$(kubectl get pods -n scdf -l app=dataflow-server -o jsonpath='{.items[0].metadata.name}')

# Port forward
kubectl port-forward -n scdf $SCDF_POD 9393:9393 &

# 建立 Stream
curl -X POST "http://localhost:9393/streams/definitions" \
  -d "name=simple-log" \
  -d "definition=http --port=8080 | log"

# 部署 Stream
curl -X POST "http://localhost:9393/streams/deployments/simple-log"
```

## 資源使用量

| 元件 | CPU Request | Memory Request |
|------|-------------|----------------|
| SCDF Server | 200m | 1Gi |
| Skipper | 200m | 1Gi |
| Kafka | 250m | 512Mi |
| MariaDB | 100m | 256Mi |
| Stream App (each) | 100m | 256Mi |

建議 Docker Desktop 配置：至少 8GB RAM, 4 CPU

## 清理環境

```bash
# 刪除 Stream
curl -X DELETE "http://localhost:9393/streams/definitions/simple-log"

# 刪除 Kind 叢集
kind delete cluster --name scdf-cluster
```

## 故障排除

### Pod 無法啟動

檢查 Pod 狀態和事件：
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

### 資料庫連線問題

確認 MariaDB 運行正常：
```bash
kubectl exec -it -n scdf $(kubectl get pods -n scdf -l app=mysql -o jsonpath='{.items[0].metadata.name}') -- mariadb -u dataflow -pdataflow123 dataflow
```

### Kafka 連線問題

測試 Kafka 連線：
```bash
kubectl run kafka-client --restart='Never' --image=apache/kafka:3.8.0 --namespace=kafka --command -- sleep infinity
kubectl exec -it kafka-client -n kafka -- /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server kafka:9092
```

## 參考資源

- [Spring Cloud Dataflow 官方文件](https://dataflow.spring.io/docs/)
- [Apache Kafka](https://kafka.apache.org/)
- [Kind 官方文件](https://kind.sigs.k8s.io/)

---

建立日期: 2026-01-30
