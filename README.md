# LLM on Kubernetes

## 项目概述

本项目记录了在 Kubernetes 集群上从零搭建 LLM（大语言模型）推理环境的完整操作流程，涵盖网络、存储、GPU 支持和模型管理所需的核心基础设施组件。

---

## 1. 镜像下载代理

配置代理以访问外网容器镜像仓库（Docker Hub、ghcr.io 等）。

### systemd 服务配置（containerd / kubelet 适用）

```ini
[Service]
Environment="HTTP_PROXY=socks5://192.168.0.101:7897"
Environment="HTTPS_PROXY=socks5://192.168.0.101:7897"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.0.0/24,::1,docker,containerd"
```

### Shell 环境变量（手动 pull 时使用）

```bash
# 设置代理
export http_proxy="http://192.168.0.101:7897"
export https_proxy="http://192.168.0.101:7897"
export no_proxy="localhost,127.0.0.1,::1"

# 取消代理
unset http_proxy
unset https_proxy
unset no_proxy
```

---

## 2. MetalLB — 负载均衡

为 LoadBalancer 类型的 Service 分配外部可访问的 IP 地址。

> 本文使用 FRR 模式，比 native 模式功能更强。

安装 MetalLB（FRR 模式）：

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-frr-k8s.yaml
```

配置 IP 地址池：

```yaml
cat >> IPAddressPool.yaml <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.0.150-192.168.0.155
EOF
```

配置 L2 广播：

```yaml
cat >> L2Advertisement.yaml <<EOF
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: my-l2-adv
  namespace: metallb-system
spec:
  ipAddressPools:
    - my-ip-pool
EOF
```

---

## 3. OpenEBS — 本地存储

提供动态 LocalPV 存储卷，用于 MinIO 等有状态服务。仅启用 local-hostpath 引擎。

```bash
helm repo add openebs https://openebs.github.io/openebs
helm repo update

helm install openebs --namespace openebs openebs/openebs \
  --set engines.replicated.mayastor.enabled=false \
  --set engines.local.zfs.enabled=false \
  --create-namespace
```

或下载到本地再安装
```bash
helm pull openebs/openebs
tar -xf openebs-4.5.0.tgz
helm install openebs --namespace openebs . \
  --set engines.replicated.mayastor.enabled=false \
  --set engines.local.zfs.enabled=false \
  --create-namespace
```

---
## 4. Prometheus - 集群监控

后续根据监控指标来扩缩容vllm服务。  

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm pull prometheus-community/prometheus
cd prometheus


# 修改配置：
vim values.yaml
server:
  persistentVolume:
    enabled: true
    statefulSetNameOverride: ""
    accessModes:
      - ReadWriteOnce
    labels: {}
    annotations: {}
    existingClaim: ""
    mountPath: /data
    size: 8Gi
    storageClass: "openebs-hostpath"
    subPath: ""


alertmanager:
  enabled: true
  persistence:
    enabled: true
    annotations: {}
    labels: {}
    storageClass: "openebs-hostpath"
    accessModes:
      - ReadWriteOnce
    size: 2Gi

helm install prometheus . -n monitoring --create-namespace
```
---

## 4. MinIO — 模型对象存储

S3 兼容的对象存储，用于存放 LLM 模型权重文件。部署为单节点模式。

### 4.1 安装

> 需要先准备 storageClass `openebs-minio-localpv`，参考 OpenEBS 章节。

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 下载 Helm values（参考配置）
wget https://raw.githubusercontent.com/iKubernetes/learning-k8s/refs/heads/master/MinIO/values.yaml
# 指定value安装
helm install minio bitnami/minio \
  --namespace minio \
  --create-namespace \
  --set auth.rootUser=minioadmin \
  --set auth.rootPassword=minioadmin \
  --set mode=standalone \
  --set persistence.size=30Gi \
  -f values.yaml


# 下载到本地再安装
helm pull bitnami/minio
tar -xf minio-5.4.0.tgz
helm install minio . \
  --namespace minio \
  --create-namespace \
  --set auth.rootUser=minioadmin \
  --set auth.rootPassword=minioadmin \
  --set mode=standalone \
  --set persistence.size=30Gi \
  --set persistence.storageClass=openebs-minio-localpv
```

### 4.2 配置 mc 客户端
下载minio客户端：
```bash
# https://minio.org.cn/docs/minio/linux/reference/minio-mc.html (各种客户端)
wget https://dl.minio.org.cn/client/mc/release/linux-amd64/mc
chmod +x mc
mv mc /usr/bin/
```

#### 创建alias
```bash
mc alias set models http://172.16.218.39:9000 admin fcdieggfjrhgk # 找不到密码可进入pod内使用env查看
# Added `models` successfully.

mc alias ls
# models
#   URL       : http://172.16.218.39:9000
#   AccessKey : admin
#   SecretKey : fcdieggfjrhgk
#   API       : s3v4
```

### 4.3 创建 Bucket

```bash
mc mb models/llm-models
# Bucket created successfully `models/llm-models`.

mc ls models
# [2026-06-17 18:52:11 CST]     0B llm-models/
```

#### 下载模型
```bash
# 创建虚拟环境（目录名可自定）
python3 -m venv modelscope_env

# 激活虚拟环境
source modelscope_env/bin/activate

# 现在再安装 modelscope
pip install modelscope


modelscope download --model Qwen/Qwen3-0.6B  --local_dir ./dir
```

### 4.4 上传模型

```bash
mc mirror --overwrite --remove ./dir/ models/llm-models/Qwen3-0.6B

mc ls models/llm-models/Qwen3-0.6B
```

---

## 5. GPU Operator — NVIDIA GPU 支持

自动部署 NVIDIA 驱动、Container Toolkit、Device Plugin、DCGM Exporter 等组件，使 K8s 集群支持 GPU 调度。
如果宿主机安装了驱动需要把 driver.enabled=true 改为 false。



```bash
# 注意：如果 nvidia-driver-daemonset-xxx 这个pod没有准备好的话，其他Pod可能会一直处于Init状态。
gpu-operator   nvidia-driver-daemonset-7vwlm    0/1     PodInitializing   0     54m


# 问题：（也可能是因为 nvidia-driver-daemonset-xxx 服务没OK的情况，多半是镜像没下载下来）
kubectl describe pods -n gpu-operator gpu-feature-discovery-tg9ps

  Warning  FailedCreatePodSandBox  4m15s                kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = unable to get OCI runtime for sandbox "4a460adf3849448a301c5bd952b0d585d39debb06c01eb774de777125b34b6c2": no runtime for "nvidia" is configured

# containerd 或 Docker 没有正确配置或识别名为 nvidia 的 OCI 运行时，因此，gpu-feature-discovery Pod 无法启动。

解决方法：
# 屏蔽 nouveau 驱动并重启。
sudo tee /etc/modprobe.d/blacklist-nouveau.conf <<< "blacklist nouveau"
sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf <<< "options nouveau modeset=0"
sudo update-initramfs -u
reboot
```
安装命令：

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=true \
  --set toolkit.enabled=true \
  --set devicePlugin.enabled=true \
  --set dcgmExporter.enabled=true \
  --set node-feature-discovery.enabled=true
```



## 模型分发：
- initContainer: (包含判断和下载) 
    OpenEBS、LocalPV、hostPath  

- Preloader: 模型预热，不管用不用，先下载好

- 模型存储感知的调度