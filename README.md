# istio-multi-cluster-aws-eks
## 说明
i.       AWS区域为海外区域
<br>ii.  EKS版本为1.30
<br>iii. 本repo步骤为Istio跨网络多集群主从模式
## 1、创建EKS集群
### 1.1 创建EC2 Linux实例
i.       创建1台EC2 Linux实例，用作操作终端
<br>ii.  创建IAM Role，并绑定AdministratorAccess策略
<br>iii. 将IAM Role绑定到该EC2实例
### 1.2 安装相关软件
i. eksctl安装
```
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
ii. kubectl安装
```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.2/2024-07-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
kubectl version --client
```
iii. istioctl安装
```
curl -sL https://istio.io/downloadIstioctl | sh -
export PATH=$HOME/.istioctl/bin:$PATH
```
### 1.3 创建EKS集群
i. 编写EKS集群配置文件
Cluster1配置
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: virgina
  region: us-east-1
  version: "1.30"

managedNodeGroups:
  - name: demo
    instanceType: m5.large
    minSize: 1
    desiredCapacity: 2
    maxSize: 4
    volumeSize: 20
    volumeType: gp3
    ssh:
      allow: true
      publicKeyName: kp_virginia
      enableSsm: true
    privateNetworking: true
EOF
```
Cluster2配置
```
cat << EOF > eks-cluster-seoul.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: seoul
  region: ap-northeast-2
  version: "1.30"

managedNodeGroups:
  - name: demo
    instanceType: m5.large
    minSize: 1
    desiredCapacity: 2
    maxSize: 4
    volumeSize: 20
    volumeType: gp3
    ssh:
      allow: true
      publicKeyName: kp_korea
      enableSsm: true
    privateNetworking: true
EOF
```
ii. 创建EKS Cluster1和Cluster2
```
eksctl create cluster -f eks-cluster-virgina.yaml
eksctl create cluster -f eks-cluster-seoul.yaml
```
等待大约20分钟，查询集群状态
```
eksctl get cluster --region ap-northeast-2
eksctl get cluster --region us-east-1
```
### 1.4 创建K8S Context
查看kubeconfig
```
kubectl config get-contexts
```
分别设置Cluster1和Cluster的K8S Context
```
export CTX_CLUSTER1=i-03b8361566ccaa46f@virgina.us-east-1.eksctl.io
export CTX_CLUSTER2=i-03b8361566ccaa46f@seoul.ap-northeast-2.eksctl.io
```
测试K8S Context
```
kubectl --context="${CTX_CLUSTER1}" get nodes
kubectl --context="${CTX_CLUSTER2}" get nodes
```
## 2、配置Isito CA证书
### 2.1 下载istio repo到本地
```
git clone https://github.com/istio/istio.git
cd istio/
```
### 2.2 生成CA证书和密钥
```
mkdir -p certs && \
  pushd certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca
make -f ../tools/certs/Makefile.selfsigned.mk cluster1-cacerts
make -f ../tools/certs/Makefile.selfsigned.mk cluster2-cacerts
```
### 2.3 创建Secret，保存证书和密钥
```
kubectl --context="${CTX_CLUSTER1}" create namespace istio-system
kubectl --context="${CTX_CLUSTER1}" create secret generic cacerts -n istio-system \
  --from-file=cluster1/ca-cert.pem \
  --from-file=cluster1/ca-key.pem \
  --from-file=cluster1/root-cert.pem \
  --from-file=cluster1/cert-chain.pem
popd

kubectl --context="${CTX_CLUSTER2}" create namespace istio-system
kubectl --context="${CTX_CLUSTER2}" create secret generic cacerts -n istio-system \
  --from-file=cluster2/ca-cert.pem \
  --from-file=cluster2/ca-key.pem \
  --from-file=cluster2/root-cert.pem \
  --from-file=cluster2/cert-chain.pem
popd
```
## 3、Istio跨网络主从架构安装
### 3.1 Cluster1主集群安装
i. 设置Cluster1网络
```
kubectl --context="${CTX_CLUSTER1}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1
```
ii. 为Cluster1创建Istio配置
```
cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF

istioctl install --set values.pilot.env.EXTERNAL_ISTIOD=true --context="${CTX_CLUSTER1}" -f cluster1.yaml
```
iii. 在Cluster1安装东西向网关
```
samples/multicluster/gen-eastwest-gateway.sh \
    --network network1 | \
    istioctl --context="${CTX_CLUSTER1}" install -y -f -
```
查看东西向网关状态
```
kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
```
iv. 开放Cluster1控制平面
```
kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f \
    samples/multicluster/expose-istiod.yaml
```

### 3.2 Cluster2从集群安装
i. 设置Cluster2网络
```
kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2
```
ii. 为Cluster2创建Istio配置
获取Cluster1东西向网关地址
```
export DISCOVERY_ADDRESS=$(kubectl \
    --context="${CTX_CLUSTER1}" \
    -n istio-system get svc istio-eastwestgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```
创建Cluste2从集群配置
```
cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: remote
  values:
    istiodRemote:
      injectionPath: /inject/cluster/cluster2/net/network2
    global:
      remotePilotAddress: ${DISCOVERY_ADDRESS}
EOF

istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
```
iii. 创建从属Secret并应用于Cluster1
```
istioctl create-remote-secret \
    --context="${CTX_CLUSTER2}" \
    --name=cluster2 | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
```
iv. 在Cluster2安装东西向网关
```
samples/multicluster/gen-eastwest-gateway.sh \
    --network network2 | \
    istioctl --context="${CTX_CLUSTER2}" install -y -f -
```
v. 开放Cluster2和Cluster2中的服务
```
kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml
```
## 4、验证主从集群安装结果
### 4.1 部署服务HelloWorld
```
# 创建命名空间
kubectl create --context="${CTX_CLUSTER1}" namespace sample
kubectl create --context="${CTX_CLUSTER2}" namespace sample

# 开启Sidecar自动注入
kubectl label --context="${CTX_CLUSTER1}" namespace sample \
    istio-injection=enabled
kubectl label --context="${CTX_CLUSTER2}" namespace sample \
    istio-injection=enabled

# 创建HelloWorld服务
kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
```
### 4.2 在Cluster1部署V1版HelloWorld
```
kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v1 -n sample
```
### 4.3 在Cluster2部署V2版HelloWorld
```
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v2 -n sample
```
### 4.4 部署Sleep
```
kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/sleep/sleep.yaml -n sample
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/sleep/sleep.yaml -n sample
```
### 4.5 验证跨集群流量
i. 从Cluster1的Sleep Pod发送请求给服务HelloWorld
```
for i in $(seq 6); do
 	kubectl exec \
 	 	--context="${CTX_CLUSTER1}" \
 	 	-n sample \
 	 	-c sleep \
 	 	"$(kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello
done
```
反复执行多次，验证HelloWorld版本在V1和V2之前切换
```
Hello version: v1, instance: helloworld-v1-69ff8fc747-s2c87
Hello version: v2, instance: helloworld-v2-779454bb5f-bvt6k
Hello version: v2, instance: helloworld-v2-779454bb5f-bvt6k
Hello version: v1, instance: helloworld-v1-69ff8fc747-s2c87
Hello version: v2, instance: helloworld-v2-779454bb5f-bvt6k
Hello version: v1, instance: helloworld-v1-69ff8fc747-s2c87
```
ii. 从Cluster2的Sleep Pod重复上一过程
```
for i in $(seq 6); do
 	kubectl exec \
 	 	--context="${CTX_CLUSTER2}" \
 	 	-n sample \
 	 	-c sleep \
 	 	"$(kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello
done
```
反复执行多次，验证HelloWorld版本在V1和V2之前切换
```
Hello version: v2, instance: helloworld-v2-779454bb5f-bvt6k
Hello version: v1, instance: helloworld-v1-69ff8fc747-s2c87
Hello version: v1, instance: helloworld-v1-69ff8fc747-s2c87
Hello version: v1, instance: helloworld-v1-69ff8fc747-s2c87
Hello version: v2, instance: helloworld-v2-779454bb5f-bvt6k
Hello version: v1, instance: helloworld-v1-69ff8fc747-s2c87
```
