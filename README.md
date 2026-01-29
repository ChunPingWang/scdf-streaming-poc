# Spring Cloud Dataflow Streaming PoC

在 Kind (Kubernetes in Docker) 上運行 Spring Cloud Dataflow 串流處理環境的 PoC 專案。

## 目錄

- [架構總覽](#架構總覽)
- [Kubernetes 叢集架構](#kubernetes-叢集架構)
- [Spring Cloud Dataflow 架構](#spring-cloud-dataflow-架構)
- [資料流架構](#資料流架構)
- [元件說明](#元件說明)
- [環境需求](#環境需求)
- [快速開始](#快速開始)
- [存取服務](#存取服務)
- [建立 Stream 範例](#建立-stream-範例)
- [故障排除](#故障排除)

---

## 架構總覽

```mermaid
graph TB
    subgraph "Local Machine"
        Browser[瀏覽器]
        CLI[kubectl / curl]
    end

    subgraph "Docker Desktop"
        subgraph "Kind Cluster: scdf-cluster"
            subgraph "Control Plane Node"
                API[Kubernetes API Server]
                ETCD[(etcd)]
                CM[Controller Manager]
                SCHED[Scheduler]
            end

            subgraph "Namespace: ingress-nginx"
                INGRESS[NGINX Ingress Controller<br/>:80, :443]
            end

            subgraph "Namespace: scdf"
                SCDF[Dataflow Server<br/>:9393 → NodePort:30080]
                SKIPPER[Skipper Server<br/>:7577 → NodePort:30081]
                MARIADB[(MariaDB<br/>:3306)]

                subgraph "Deployed Stream Apps"
                    APP1[Source App]
                    APP2[Processor App]
                    APP3[Sink App]
                end
            end

            subgraph "Namespace: kafka"
                KAFKA[Apache Kafka<br/>:9092 → NodePort:30092]
            end
        end
    end

    Browser -->|:30080| SCDF
    Browser -->|:80| INGRESS
    CLI -->|:30080, :30081| API

    INGRESS --> SCDF
    SCDF <--> SKIPPER
    SCDF --> MARIADB
    SKIPPER --> MARIADB
    SKIPPER -.->|部署/管理| APP1
    SKIPPER -.->|部署/管理| APP2
    SKIPPER -.->|部署/管理| APP3

    APP1 -->|Message| KAFKA
    KAFKA -->|Message| APP2
    APP2 -->|Message| KAFKA
    KAFKA -->|Message| APP3

    style SCDF fill:#6db33f,color:#fff
    style SKIPPER fill:#6db33f,color:#fff
    style KAFKA fill:#231f20,color:#fff
    style MARIADB fill:#c76e00,color:#fff
    style INGRESS fill:#009639,color:#fff
```

---

## Kubernetes 叢集架構

```mermaid
graph TB
    subgraph "Kind Cluster Architecture"
        subgraph "scdf-cluster-control-plane"
            subgraph "System Components"
                COREDNS[CoreDNS<br/>DNS 解析服務]
                KINDNET[Kindnet<br/>CNI 網路插件]
                PROXY[Kube-Proxy<br/>服務代理]
            end

            subgraph "Control Plane"
                APISERVER[API Server<br/>叢集入口點]
                ETCD[(etcd<br/>叢集狀態儲存)]
                CONTROLLER[Controller Manager<br/>資源控制器]
                SCHEDULER[Scheduler<br/>Pod 調度器]
            end
        end

        subgraph "Port Mappings"
            P80[Host :80 → Container :80]
            P443[Host :443 → Container :443]
            P30080[Host :30080 → Container :30080]
            P30081[Host :30081 → Container :30081]
            P30092[Host :30092 → Container :30092]
        end
    end

    APISERVER --> ETCD
    CONTROLLER --> APISERVER
    SCHEDULER --> APISERVER
    COREDNS --> APISERVER
    PROXY --> APISERVER

    style APISERVER fill:#326ce5,color:#fff
    style ETCD fill:#419eda,color:#fff
    style CONTROLLER fill:#326ce5,color:#fff
    style SCHEDULER fill:#326ce5,color:#fff
```

### Namespace 配置

```mermaid
graph LR
    subgraph "Kubernetes Namespaces"
        subgraph "kube-system"
            KS1[CoreDNS]
            KS2[Kindnet]
            KS3[Kube-Proxy]
        end

        subgraph "ingress-nginx"
            IN1[Ingress Controller]
            IN2[Admission Webhook]
        end

        subgraph "scdf"
            S1[Dataflow Server]
            S2[Skipper]
            S3[MariaDB]
            S4[Stream Apps...]
        end

        subgraph "kafka"
            K1[Kafka Broker]
        end
    end

    style scdf fill:#e8f5e9
    style kafka fill:#fff3e0
    style ingress-nginx fill:#e3f2fd
```

---

## Spring Cloud Dataflow 架構

```mermaid
graph TB
    subgraph "Spring Cloud Dataflow 核心架構"
        subgraph "Dataflow Server"
            DSL[Stream DSL Parser<br/>解析串流定義]
            REST[REST API<br/>:9393]
            DASHBOARD[Dashboard UI<br/>Web 管理介面]
            APP_REG[App Registry<br/>應用程式註冊表]
        end

        subgraph "Skipper Server"
            DEPLOYER[Kubernetes Deployer<br/>部署管理器]
            RELEASE[Release Manager<br/>版本管理]
            MANIFEST[Manifest Service<br/>部署清單服務]
        end

        subgraph "Metadata Store"
            DB[(MariaDB<br/>元資料儲存)]
        end

        subgraph "Message Broker"
            KAFKA_B[Apache Kafka<br/>訊息佇列]
        end

        subgraph "Kubernetes Platform"
            K8S_API[Kubernetes API]

            subgraph "Stream Application Pods"
                POD1[Source Pod]
                POD2[Processor Pod]
                POD3[Sink Pod]
            end
        end
    end

    DASHBOARD --> REST
    REST --> DSL
    REST --> APP_REG
    REST -->|Stream 部署請求| DEPLOYER

    DEPLOYER --> RELEASE
    RELEASE --> MANIFEST
    MANIFEST -->|建立 Deployment/Service| K8S_API

    DSL --> DB
    APP_REG --> DB
    RELEASE --> DB

    K8S_API --> POD1
    K8S_API --> POD2
    K8S_API --> POD3

    POD1 -->|Publish| KAFKA_B
    KAFKA_B -->|Subscribe| POD2
    POD2 -->|Publish| KAFKA_B
    KAFKA_B -->|Subscribe| POD3

    style DASHBOARD fill:#6db33f,color:#fff
    style REST fill:#6db33f,color:#fff
    style DEPLOYER fill:#6db33f,color:#fff
    style KAFKA_B fill:#231f20,color:#fff
    style DB fill:#c76e00,color:#fff
```

### SCDF 元件互動流程

```mermaid
sequenceDiagram
    participant User as 使用者
    participant UI as Dashboard UI
    participant DS as Dataflow Server
    participant SK as Skipper
    participant K8S as Kubernetes API
    participant DB as MariaDB
    participant KF as Kafka

    User->>UI: 1. 建立 Stream 定義
    UI->>DS: 2. POST /streams/definitions
    DS->>DB: 3. 儲存 Stream 定義

    User->>UI: 4. 部署 Stream
    UI->>DS: 5. POST /streams/deployments/{name}
    DS->>SK: 6. 部署請求
    SK->>DB: 7. 建立 Release 記錄
    SK->>K8S: 8. 建立 Deployment & Service
    K8S-->>SK: 9. 部署狀態回報
    SK-->>DS: 10. 部署完成

    Note over K8S,KF: Stream Apps 開始運行
    K8S->>KF: 11. Source 發送訊息
    KF->>K8S: 12. Processor 接收處理
    K8S->>KF: 13. Processor 發送結果
    KF->>K8S: 14. Sink 接收並輸出
```

---

## 資料流架構

### Stream Pipeline 概念

```mermaid
graph LR
    subgraph "Stream Pipeline"
        subgraph "Source"
            S[HTTP Source<br/>接收外部請求]
        end

        subgraph "Processor"
            P1[Filter<br/>過濾資料]
            P2[Transform<br/>轉換格式]
        end

        subgraph "Sink"
            SK[Log Sink<br/>輸出日誌]
        end
    end

    subgraph "Kafka Topics"
        T1[topic: stream.http]
        T2[topic: stream.filter]
        T3[topic: stream.transform]
    end

    S -->|produce| T1
    T1 -->|consume| P1
    P1 -->|produce| T2
    T2 -->|consume| P2
    P2 -->|produce| T3
    T3 -->|consume| SK

    style S fill:#4caf50,color:#fff
    style P1 fill:#2196f3,color:#fff
    style P2 fill:#2196f3,color:#fff
    style SK fill:#f44336,color:#fff
    style T1 fill:#231f20,color:#fff
    style T2 fill:#231f20,color:#fff
    style T3 fill:#231f20,color:#fff
```

### 即時交易風控範例架構

```mermaid
graph LR
    subgraph "External"
        CLIENT[交易系統]
    end

    subgraph "Stream: fraud-detection"
        HTTP[HTTP Source<br/>:8080<br/>接收交易資料]
        FILTER[Filter Processor<br/>過濾大額交易<br/>amount > 1000]
        SCORER[Risk Scorer<br/>風險評分計算]
        LOG[Log Sink<br/>告警輸出]
    end

    subgraph "Kafka Topics"
        T1((transactions))
        T2((filtered))
        T3((scored))
    end

    subgraph "Alert System"
        ALERT[告警通知]
    end

    CLIENT -->|POST /transactions| HTTP
    HTTP --> T1
    T1 --> FILTER
    FILTER --> T2
    T2 --> SCORER
    SCORER --> T3
    T3 --> LOG
    LOG -.-> ALERT

    style HTTP fill:#4caf50,color:#fff
    style FILTER fill:#ff9800,color:#fff
    style SCORER fill:#f44336,color:#fff
    style LOG fill:#9c27b0,color:#fff
```

---

## 元件說明

### 核心元件

| 元件 | 說明 | Port | 用途 |
|------|------|------|------|
| **Dataflow Server** | SCDF 主服務 | 9393 (NodePort: 30080) | 提供 REST API 和 Dashboard UI，管理 Stream/Task 定義 |
| **Skipper Server** | 部署管理服務 | 7577 (NodePort: 30081) | 負責 Stream 應用程式的部署、升級和回滾 |
| **MariaDB** | 關聯式資料庫 | 3306 | 儲存 SCDF 元資料、Stream 定義、部署歷史 |
| **Apache Kafka** | 訊息佇列 | 9092 (NodePort: 30092) | Stream 應用程式之間的訊息傳遞 |
| **NGINX Ingress** | 入口控制器 | 80, 443 | 提供 HTTP/HTTPS 路由 |

### Stream 應用程式類型

```mermaid
graph TB
    subgraph "Application Types"
        subgraph "Source 來源"
            S1[HTTP - 接收 HTTP 請求]
            S2[Kafka - 消費 Kafka 訊息]
            S3[JDBC - 查詢資料庫]
            S4[File - 讀取檔案]
            S5[TCP - 接收 TCP 連線]
        end

        subgraph "Processor 處理器"
            P1[Filter - 過濾資料]
            P2[Transform - 轉換格式]
            P3[Splitter - 分割訊息]
            P4[Aggregator - 聚合資料]
            P5[Script - 自訂腳本]
        end

        subgraph "Sink 接收"
            K1[Log - 輸出日誌]
            K2[JDBC - 寫入資料庫]
            K3[Kafka - 發送到 Kafka]
            K4[File - 寫入檔案]
            K5[HTTP - 發送 HTTP 請求]
        end
    end

    style S1 fill:#4caf50,color:#fff
    style S2 fill:#4caf50,color:#fff
    style S3 fill:#4caf50,color:#fff
    style S4 fill:#4caf50,color:#fff
    style S5 fill:#4caf50,color:#fff
    style P1 fill:#2196f3,color:#fff
    style P2 fill:#2196f3,color:#fff
    style P3 fill:#2196f3,color:#fff
    style P4 fill:#2196f3,color:#fff
    style P5 fill:#2196f3,color:#fff
    style K1 fill:#f44336,color:#fff
    style K2 fill:#f44336,color:#fff
    style K3 fill:#f44336,color:#fff
    style K4 fill:#f44336,color:#fff
    style K5 fill:#f44336,color:#fff
```

---

## 環境需求

| 工具 | 版本要求 | 說明 |
|------|----------|------|
| Docker Desktop | 4.x+ | 容器運行環境，至少配置 8GB RAM, 4 CPU |
| Kind | v0.20+ | Kubernetes in Docker |
| kubectl | v1.28+ | Kubernetes CLI |
| Helm | v3.12+ | Kubernetes 套件管理 |

### 資源需求

| 元件 | CPU Request | Memory Request | CPU Limit | Memory Limit |
|------|-------------|----------------|-----------|--------------|
| Dataflow Server | 200m | 1Gi | 1000m | 1536Mi |
| Skipper | 200m | 1Gi | 1000m | 1536Mi |
| Kafka | 250m | 512Mi | 500m | 1Gi |
| MariaDB | 100m | 256Mi | 500m | 512Mi |
| Stream App (each) | 100m | 256Mi | 500m | 512Mi |

---

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
kubectl wait --for=condition=ready pod -l app=kafka -n kafka --timeout=180s
```

### 4. 部署 MariaDB

```bash
kubectl apply -f mysql.yaml
kubectl wait --for=condition=ready pod -l app=mysql -n scdf --timeout=120s
```

### 5. 部署 Spring Cloud Dataflow

```bash
kubectl apply -f scdf.yaml

# 等待 Skipper 就緒（約 3-5 分鐘）
kubectl wait --for=condition=ready pod -l app=skipper -n scdf --timeout=300s

# 等待 Dataflow Server 就緒（約 3-5 分鐘）
kubectl wait --for=condition=ready pod -l app=dataflow-server -n scdf --timeout=300s
```

### 6. 安裝 Ingress Controller（選用）

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=180s
kubectl apply -f ingress.yaml
```

---

## 存取服務

| 服務 | URL | 說明 |
|------|-----|------|
| SCDF Dashboard | http://localhost:30080/dashboard | Web 管理介面 |
| Dataflow REST API | http://localhost:30080 | REST API 端點 |
| Skipper | http://localhost:30081 | Skipper 管理 API |
| Kafka (外部) | localhost:30092 | Kafka broker |

---

## 建立 Stream 範例

### 方法一：使用 Dashboard UI

1. 開啟 http://localhost:30080/dashboard
2. 點擊 **Apps** → **Add Application(s)**
3. 選擇 **Import application starters from dataflow.spring.io**
4. 選擇 **Stream application starters for Kafka/Maven**
5. 點擊 **Import**
6. 進入 **Streams** → **Create Stream**
7. 輸入 Stream DSL：`http --port=8080 | log`
8. 點擊 **Create Stream** 並命名為 `simple-log`
9. 點擊 **Deploy**

### 方法二：使用 REST API

```bash
# 匯入預建應用程式
curl -X POST "http://localhost:30080/apps" \
  -d "uri=https://dataflow.spring.io/kafka-maven-latest"

# 建立 Stream 定義
curl -X POST "http://localhost:30080/streams/definitions" \
  -d "name=simple-log" \
  -d "definition=http --port=8080 | log"

# 部署 Stream
curl -X POST "http://localhost:30080/streams/deployments/simple-log"

# 檢查部署狀態
curl "http://localhost:30080/streams/definitions/simple-log"
```

### 測試 Stream

```bash
# 取得 HTTP Source Pod 並進行 port forward
HTTP_POD=$(kubectl get pods -n scdf -l spring-app-id=simple-log-http -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n scdf $HTTP_POD 8080:8080 &

# 發送測試訊息
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello SCDF!"}'

# 查看 Log Sink 輸出
LOG_POD=$(kubectl get pods -n scdf -l spring-app-id=simple-log-log -o jsonpath='{.items[0].metadata.name}')
kubectl logs -f -n scdf $LOG_POD
```

---

## 檔案說明

| 檔案 | 說明 |
|------|------|
| `kind-config.yaml` | Kind 叢集配置，定義 port mapping 和節點設定 |
| `kafka-cluster.yaml` | Apache Kafka 部署配置（KRaft 模式，無 Zookeeper） |
| `mysql.yaml` | MariaDB 資料庫部署配置 |
| `scdf.yaml` | Spring Cloud Dataflow Server 和 Skipper 完整配置 |
| `scdf-values.yaml` | SCDF Helm values 參考檔案 |
| `ingress.yaml` | NGINX Ingress 路由配置 |

---

## 故障排除

### 常見問題

#### Pod 無法啟動

```bash
# 檢查 Pod 狀態
kubectl get pods -n scdf
kubectl describe pod <pod-name> -n scdf

# 查看 Pod 日誌
kubectl logs <pod-name> -n scdf
kubectl logs <pod-name> -n scdf --previous  # 查看前一個容器的日誌
```

#### 資料庫連線問題

```bash
# 測試 MariaDB 連線
kubectl exec -it -n scdf $(kubectl get pods -n scdf -l app=mysql -o jsonpath='{.items[0].metadata.name}') \
  -- mariadb -u dataflow -pdataflow123 dataflow -e "SHOW TABLES;"
```

#### Kafka 連線問題

```bash
# 建立測試 Pod
kubectl run kafka-client --restart='Never' --image=apache/kafka:3.8.0 \
  --namespace=kafka --command -- sleep infinity

# 列出 Topics
kubectl exec -it kafka-client -n kafka -- \
  /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server kafka:9092

# 刪除測試 Pod
kubectl delete pod kafka-client -n kafka
```

#### SCDF 健康檢查

```bash
# 檢查 Dataflow Server
curl http://localhost:30080/about

# 檢查 Skipper
curl http://localhost:30081/api/about

# 檢查已部署的 Streams
curl http://localhost:30080/streams/definitions
```

---

## 清理環境

```bash
# 刪除所有 Streams
curl -X DELETE "http://localhost:30080/streams/definitions/simple-log"

# 刪除 Kind 叢集
kind delete cluster --name scdf-cluster
```

---

## 參考資源

- [Spring Cloud Dataflow 官方文件](https://dataflow.spring.io/docs/)
- [Spring Cloud Dataflow Reference Guide](https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kind - Kubernetes in Docker](https://kind.sigs.k8s.io/)
- [Stream Application Starters](https://cloud.spring.io/spring-cloud-stream-app-starters/)

---

> **建立日期**: 2026-01-30
> **環境版本**: SCDF 2.11.5 / Kafka 3.8.0 / Kind v0.27.0 / Kubernetes v1.32.2
