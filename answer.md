# 透過 Prometheus 監控 k8s cluster
## Preparation
* 安裝 kind
* 安裝 docker
* 安裝 kubectl
## 建立 k8s cluster
這邊使用 kind 官網提供的範例 yaml 來做修改，目的是建立 3 個 master node 與 4 個 worker node 。
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