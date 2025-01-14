# 透過 Prometheus 監控 k8s cluster
## Preparation
* 安裝 kind
* 安裝 docker
* 安裝 kubectl

## 建立 k8s cluster
這邊使用 kind 官網提供的範例 yaml 來做修改，目的是建立 3 個 master node 與 4 個 worker node 。
由於這邊我不知道為什麼無論如何都建不了 5 個以上的 node ，所以只建 1 個 master node 。

[cluster.yml](https://github.com/SonixChang/k8s_test/blob/main/kind/cluster/cluster.yml)
```kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
kubeadmConfigPatches:
- |
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration
  evictionHard:
    nodefs.available: "0%"
kubeadmConfigPatchesJSON6902:
- group: kubeadm.k8s.io
  version: v1beta2
  kind: ClusterConfiguration
  patch: |
    - op: add
      path: /apiServer/certSANs/-
      value: my-hostname
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
- role: worker
```

再輸入下面指令建立 cluster 。
```
kind create cluster --config cluster.yaml
```

再輸入以下指令檢查 node 。
```
kubectl get no
```
![Image text](http://assets/nodes.png)
## Node 分角色
幫 worker node 分成兩個 infra node 與兩個 application node 。這邊用的方法使用 taint 。

為節點 kind-worker 與 kind-worker2 加上 taint ， role=infra:NoSchedule 。 
```
kubectl taint no kind-worker role=infra:NoSchedule
kubectl taint no kind-worker2 role=infra:NoSchedule
```

為節點 kind-worker3 與 kind-worker4 加上 taint ， role=app:NoSchedule 。 
```
kubectl taint no kind-worker3 role=app:NoSchedule
kubectl taint no kind-worker4 role=app:NoSchedule
```
## 安裝 MetalLB ，以 L2 模式安裝
這邊就依照[官網](https://metallb.universe.tf/installation/)安裝。
### Preparation
如果 kube-proxy 使用 ipvs mode ，要啟用 strict arp mode 。
### 安裝
官網提供三種裝法。這邊使用最基本的 yaml 安裝。
#### 修改 yaml 文件
由於要將 speaker 佈署在 infra node 上，所以必須在 speaker 的 pod 上加上 tolerations 。
```
tolerations:
- effect: NoSchedule
  key: role
  operator: Equal
  value: infra
```
#### 執行
```
kubectl apply -f metallb-native.yaml
kubectl apply -f metallb-frr.yaml
```
### 建立 IP 池
L2advertisement.yml
```
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```
ip-pool.yml
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.0.140-172.18.0.150
```
輸入以下指令建立
```
kubectl create -f  L2advertisement.yml ip-pool.yml
```

* 為什麼這邊 ip 池的範圍要設定成這樣，是因為 kind 在建立 cluster 時，會在 docker 內建立一個 kind 專屬的 nework bridge ，而這個網段是 172.18.0.0/16 ，所以 ip 池必須設定在這個網段範圍內，本機 (cluster 外)才可以連到該 ip 。

### 測試
建立 deployment 與 LoadBalancer 測試。

nginx-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      tolerations:
      - key: role
        operator: Equal
        value: app
        effect: NoSchedule
      containers:
      - name: nginx
        image: nginx:1.27.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```
nginx-lb.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: nginx-port
    protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```
執行以下命令檢查 LoadBalancer ，確定有拿到 ip 池內的 ip 。
```
kubectl get svc
```
![Image text](http://assets/loadbalancer.png)

IP 是 172.18.0.141:80 ，執行以下命令確定請求得到回應。
```
curl 172.18.0.141:80
```

## 安裝 Prometheus, kube-state-metrics 在 infra node 上，安裝 node exporter 在每個 node 上

### 安裝
我們選擇用 [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus/tree/main/manifests) 來安裝，裡面檔案很多所以先做分類，分類後會有以下目錄與檔案。
* alertmanager
* blackboxExporter
* grafana
* kubernetesControlPlane
* kubeStateMetrics
* nodeExporter
* prometheus
* prometheusAdapter
* prometheusOperator
* setup
* kubePrometheus-prometheusRole.yaml

由於 prometheus 與 kube-state-metrics 要佈署在 infra node 上，所以一樣要加上 tolerations ，而 node exporter 需要使用 daemonset 來建立，正好 kube-prometheus 內的 node exporter 就是用 daemonset ，所以不用修改。剩下的 alertmanager, blackboxExporter 不會用到， grafana 要佈署在 cluster 外，所以這些就不佈署了，其他的依序佈署。
```
kubectl apply -f setup
kubectl apply -f kubernetesControlPlane
kubectl apply -f kubeStateMetrics
kubectl apply -f nodeExporter
kubectl apply -f prometheus
kubectl apply -f prometheusAdapter
kubectl apply -f prometheusOperator
kubectl apply -f kubePrometheus-prometheusRole.yaml
```

這些 resource 都會被建立在 monitoring 的 namespace 。輸入以下指令確認
```
kubectl get all -n monitoring
```
### 刪除 Network Policy
這時可以使用 port-forward 的方式存取 prometheus 了，以及查看 metrics 。
prometheus 預設的 port 是 9090 。
```
kubectl port-forward prometheus-k8s-0 -n monitoring 9090 --address ${Vm IP}
```

但是因為 NetworkPolicy 的關係，導致我們只能使用 Ingress 的方式存取 prometheus 。
輸入以下命令查看 NetworkPolicy
```
kubectl get networkpolicy -n monitoring
```
會發現 prometheus-k8s NetworkPolicy 限制了 prometheus 只能被 ingress 存取。
因為這邊我想用 Loadbalancer 存取，所以直接刪除該 NetworkPolicy 。

prometheus-lb.yml
```
apiVersion: v1
kind: Service
metadata:
  name: prometheus-lb
  namespace: monitoring
spec:
  selector:
    app.kubernetes.io/name: prometheus
  ports:
  - name: web
    protocol: TCP
    port: 9090
    targetPort: 9090
  type: LoadBalancer
```

直接 create 
```
kubectl create -f prometheus-lb.yml
```
輸入以下指令查看
```
kubectl get svc -n monitoring
```
![Image text](http://assets/prometheus-lb.png)
可以看到一樣取得了 ip 172.18.0.142，發送請求測試
```
curl 172.18.0.142:9090
```

## 安裝 Grafana
安裝在本機 docker 上。輸入以下命令
```
docker run -idt -p 3000:3000 --name=grafana --network=kind --volume grafana-storage:/var/lib/grafana --restart=always grafana/grafana-oss
```
