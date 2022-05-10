# Kubernetes-EFK

這是我們 Kubernetes 文章的第三篇，此文章會紀錄我將 EFK 建在 Kubernetes 上面，最後也會統整實作過程中，可能會遇到的一些問題，讓大家在學習時，可以更有效率，不用 debug 到死 😎😎

<br>

那由於本文會直接帶入程式，觀念部分，可以先查看：

* Kubernetes : [Kubernetes (K8s) 介紹 - 基本](https://pin-yi.me/k8s/)
*  kubernetes : [Kubernetes (K8s) 介紹 - 進階 (Service、Ingress、StatefulSet、Deployment、ReplicaSet)](https://pin-yi.me/k8s-advanced/)
*  EFK : [用 EFK 收集容器日誌 (HAProxy、Redis Sentinel、Docker-compose)](https://pin-yi.me/docker-compose-redis-sentinel-haproxy-efk/)

<br>

此文章程式碼也會同步到 Github ，需要的也可以去查看歐！要記得先確定一下自己的版本 [Github 程式碼連結](https://github.com/880831ian/Kubernetes-EFK) 😆

## 版本資訊

* macOS：11.6
* Minikube：v1.25.2
* hyperkit：0.20200908
* Kubectl：Client Version：v1.22.5、Server Version：v1.23.3
* Elasticsearch：8.1.3
* Fluentd：v1.14.6-debian-elasticsearch7-1.0
* Kibana：8.1.3

<br>


## 實作

### 創建命名空間

在我們開始之前，我們首先建立一個命名空間，我們會將 EFK 所有的工具都安裝在此。那我想創建一個名為 `kube-logging` 的 namespace，先查詢現有的命名空間是否有重複：

```sh
$ kubectl get namespaces
```

<br>

我們可以看到這些已經存在的 namespace ：

```sh
NAME                   STATUS   AGE
default                Active   92m
kube-logging           Active   89m
kube-node-lease        Active   92m
kube-public            Active   92m
kube-system            Active   92m
kubernetes-dashboard   Active   92m
```

<br>

那要怎麼創建一個命名空間呢？先打開編輯器編輯名為 `kube-logging.yaml`：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-logging
```

<br>

完成後，使用 `apply` 來創建命名空間：

```sh
$ kubectl apply -f kube-logging.yaml

namespace/kube-logging created
```

<br>

再使用 `kubectl get namespaces` 查看是否多了一個名為 `kube-logging` 的命名空間：

```sh
$ kubectl get namespaces

NAME                   STATUS   AGE
default                Active   2m23s
kube-logging           Active   88s
kube-node-lease        Active   2m24s
kube-public            Active   2m24s
kube-system            Active   2m24s
kubernetes-dashboard   Active   2m19s
```

<br>

### 創建 Elasticsearch StatefulSet

我們已經創建好一個命名空間來放我們的 EFK，首先先部署副本數有 3 個的 Elasticsearch Pod。為什麼要使用 3 個 呢？使用 3 個 Elasticsearch Pod 是為了避免在高可用性、多節點叢集時出現錯誤，當其中一個 Elasticsearch Pod 故障，其他 2 個就會選舉後來接替，保證叢集可以繼續運行。

<br>

#### 創建 Headless Service

我們先創建名為 `elasticsearch_svc.yaml` 的 yaml 檔，用來處理 service 的問題：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: kube-logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```
我們有創建一個命名空間，所以要先在 metadata 加入 namespace: kube-logging。記得要設定標籤，當我們將 Elasticsearch StatefulSet 與此 Service 關聯時，Service 會返回指向的帶有標籤的 Elasticsearch Pod。然後我們設置 `clusterIP: None` 定義該 Service 為 Headless Service。最後定義 Port 9200 為 REST API、Port 9300 為 Node 之間的通信。

<br>

一樣使用 `apply` 來建立 Service：

```sh
$ kubectl apply -f elasticsearch-svc.yaml

service/elasticsearch created
```

我們這次直接查看 minikube dashboard：

<br>

![圖片](https://raw.githubusercontent.com/880831ian/Kubernetes-EFK/master/images/service.png)

<br>

#### 創建 StatefulSet

創建名為 `elasticsearch_statefulset.yaml` 的 yaml 檔案 (因為程式長度，所以分開說明，要完整請參考 [Github 程式碼連結](https://github.com/880831ian/Kubernetes-EFK))：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: kube-logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
```        
 名稱取名為 `es-cluster` ，我一樣使用 kube-logging 的 namespace，設定 ServiceName `elasticsearch` 是確保 StatefulSet 中的每一個 Pod 都可以使用以下 DNS 位址進行訪問：`es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local` ，其中 `[0,1,2]` 對應 Pod 分配的整數序號。

我們指定副本數為 3，並將 `matchLabels` 選擇器設定 `app: elasticseach`，我們在將其鏡像到該 `.spec.template.metadata`，`.spec.selector.matchLabels` 跟 `.spec.template.metadata.labels` 必須相同。

<br>

接續...
```yaml
    spec:
      containers:
        - name: elasticsearch
          image: elasticsearch:8.1.3
          resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
          ports:
            - containerPort: 9200
              name: rest
              protocol: TCP
            - containerPort: 9300
              name: inter-node
              protocol: TCP
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          env:
            - name: cluster.name
              value: k8s-logs
            - name: node.name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: discovery.seed_hosts
              value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
            - name: cluster.initial_master_nodes
              value: "es-cluster-0,es-cluster-1,es-cluster-2"
            - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
            - name: xpack.security.enabled # 記得要加上，因爲 Elasticsearch 8.x版本後會自動開啟SSL，如果沒有設定他就會一直重新啟動
              value: "false"
```

我們定義容器名稱是 `elasticsearch`，映像檔是 `elasticsearch:8.1.3` ，要記得這個版本要與後面的 kibana 相同，我們在 resources 設置容器至少需要 0.1 CPU，最多可以到 1 個 CPU。一樣設定 Port 9200 為 REST API、Port 9300 為 Node 之間的通信，並將名為 `data` 的PersistentVolume 的容器掛載到容器的 `/usr/share/elasticsearch/data`。

最後幫容器設置一些環境變數：
* `cluster.name`：Elasticsearch 叢集的名稱，我們設定為 `k8s-logs`。
* `node.name`：節點名稱，我們設置 valueFrom 使用 `.metadata.name`，它會解析為 `es-cluster-[0,1,2]`，取決節點的指定順序。
* `discovery.seed_hosts`：設置叢集中主節點列表，這些節點將會為節點發現過程中提供 Pod，但由於我們配置的 Headless Service，所以我們的 Pod 具有 `es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local` Kubernetes DNS 解析。
* `cluster.initial_master_nodes`：這邊指定將參與主節點的選舉過程節點列表，這邊是通過節點 `node.name` 來辨識，不是透過主機名。
* `ES_JAVA_OPTS`：這邊我們設置 `-Xms512m -Xmx512m`，告訴 JVM 使用最大跟最小 512 MB，可以依據資源來做調配。
* `xpack.security.enabled`：這是也是我在 elasticsearch 卡很久的一個設定，詳細的會在最後的常見問題中提到，大家只要記得 elasticsearch 8.x 以後都需要多這個。

<br>

接續...
```yaml
      initContainers:
        - name: fix-permissions
          image: busybox
          command:
            ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          securityContext:
            privileged: true
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
        - name: increase-vm-max-map
          image: busybox
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
        - name: increase-fd-ulimit
          image: busybox
          command: ["sh", "-c", "ulimit -n 65536"]
          securityContext:
            privileged: true
```
我們這個區塊定義了主容器運行前的初始化設定：
* `fix-permissions`：因為默認情況下，Kubernetes 會將數據目錄掛載為 `root`，導致 Elasticsearch 無法訪問，所以才會多這個運行 `chown` 將 Elasticsearch 數據目錄的所有者和組更改為 `1000:1000 /usr/share/elasticsearch/data`。詳細可以參考 [Notes for Production Use and Defaults](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_notes_for_production_use_and_defaults)。
* `increase-vm-max-map`：這邊是因為默認情況下內存會太低，所以多這個用 `sysctl -w` 來調整內存大小。詳細可以參考 [Elasticsearch documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html)。
* `iincrease-fd-ulimit`：它運行 `ulimit` 調整打開文件描述的最大數量。詳細可以參考 [Notes for Production Use and Defaults](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_notes_for_production_use_and_defaults)。

<br>

接續...
```yaml
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: elasticsearch
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 3Gi
```
這邊定義了 StatefulSet 的 `volumeClaimTemplates`。Kubernetes 會幫 Pod 創建 PersistentVolume，我們在上面的命名將它取為 `data`，並與 `app: elasticsearch`  StatefulSet 相同標籤。
我們將訪問的模式設定為 `ReadWriteOnce`，代表我們只能被單個節點以讀寫方式掛載，最後我們設定每個 PersistentVolume 的大小為 3GiB。

<br>

都完成後，我們一樣用 `apply` 部署 StatefulSet：

```
$ kubectl apply -f elasticsearch-statefulset.yaml 

statefulset.apps/es-cluster created
```

<br>

我們可以查看 minikube dashboard：

<br>

![圖片](https://raw.githubusercontent.com/880831ian/Kubernetes-EFK/master/images/statefulset.png)

<br>

可以看到已經成功建好，接著我們使用 port-forward 來測試是否正常運作：

```
$ kubectl port-forward es-cluster-0 9200:9200 --namespace=kube-logging
$ curl http://localhost:9200/

{
  "name" : "es-cluster-0",
  "cluster_name" : "k8s-logs",
  "cluster_uuid" : "aGxBystkQFW-xvjJ98Pxcw",
  "version" : {
    "number" : "8.1.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "39afaa3c0fe7db4869a161985e240bd7182d7a07",
    "build_date" : "2022-04-19T08:13:25.444693396Z",
    "build_snapshot" : false,
    "lucene_version" : "9.0.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
如果像我上面一樣有跳出 ”You Know, for Search“ 就代表 Elasticsearch 已經在正常運作囉 🥳

<br>

## 參考資料

[How To Set Up an Elasticsearch, Fluentd and Kibana (EFK) Logging Stack on Kubernetes](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes)
