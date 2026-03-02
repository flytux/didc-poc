# RKE2와 Rancher를 이용한 AIOPS 환경 구성

## 1. GPU 클러스터 구성
1. RKE2 최신 버전 설치 (Master - Worker)
2. GPU Operator 설치
3. Metallb 설치
4. NFS 스토리지 클래스 설치

### GPU Operator 설치
```bash
kubectl apply -f gpu-operator.yaml
```

### Metallb 설치
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml 
kubectl apply -f metallb-config.yaml
```

### NFS 스토리지 클래스 설치
```bash
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.5.0/deploy/install-driver.sh | bash -s v4.5.0 --
kubectl apply -f nfs-sc.yaml
```

## 2. 클러스터 관리 환경 구성
1. Rancher Manager 설치
2. Harbor 설치
3. Fleet 배포 테스트

### cert-manager 설치
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml
```

### Rancher 설치
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest 

helm upgrade -i rancher rancher-latest/rancher \
  --set hostname=rancher.local --set bootstrapPassword=admin \
  --set replicas=1 --set global.cattle.psp.enabled=false \
  --create-namespace -n cattle-system
```

### Harbor 설치
Helm Chart Repository 추가:
```
https://helm.goharbor.io
```

사설인증서 노드 별 추가:
```bash
kubectl create ns harbor
kubectl create secret tls harbor-ingress --key harbor.key --cert harbor.crt -n harbor

# Ubuntu
cp harbor.crt harbor.key /usr/local/share/ca-certificates/ 
update-ca-certificates
```

### Fleet 배포 테스트
```bash
kubectl apply -f fleet-simple.yaml
```

## 3. AIOPS 구성
1. Istio 설치
2. Knative 설치
3. KServe 설치

### Gateway API CRD 설치
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

### Istio 설치
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts --force-update

helm install istio-base istio/base -n istio-system --wait --set defaultRevision=default --create-namespace --version 1.27.1

helm install istiod istio/istiod -n istio-system --wait --version 1.27.1 \
   --set proxy.autoInject=disabled \
   --set-string pilot.podAnnotations."cluster-autoscaler\.kubernetes\.io/safe-to-evict"=true

helm install istio-ingressgateway istio/gateway -n istio-system --version 1.27.1 \
   --set-string podAnnotations."cluster-autoscaler\.kubernetes\.io/safe-to-evict"=true
```

### Knative 설치
```bash
helm install knative-operator --namespace knative-serving --create-namespace --wait \
      https://github.com/knative/operator/releases/download/knative-v1.15.7/knative-operator-v1.15.7.tgz

kubectl apply -f kserving.yaml
```

### KServe 설치
```bash
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd --version v0.16.0 --namespace kserve --create-namespace --wait

helm install kserve oci://ghcr.io/kserve/charts/kserve --version v0.16.0 --namespace kserve --create-namespace --wait --set-string kserve.controller.deploymentMode="Serverless"

kubectl get pods -n kserve
```

## 4. ML서비스 테스트
1. KServe 런타임 설치
2. Inference Service 생성
3. Inference Service 호출 (IRIS)

### KServe 런타임 설치
```bash
kubectl create ns kserv-test
kubectl apply -f serving-runtime.yaml
kubectl get servingruntime.serving.kserve.io -n kserv-test
```

### Inference Service 생성
```bash
kubectl apply -f inference-service.yaml
kubectl get inferenceservices sklearn-iris -n kserve-test
```

### Inference Service 호출 (IRIS)
```bash
# Load Balancer IP 확인
LB_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Inference Service 호출 (테스트 1)
curl -v -H "Host: sklearn-iris.kserve-test.didc.local" -H "Content-Type: application/json" \
  http://$LB_IP/v1/models/sklearn-iris:predict -d @./iris-input.json

# Inference Service 호출 (테스트 2)
curl -v -H "Host: sklearn-iris.kserve-test.didc.local" -H "Content-Type: application/json" \
  http://$LB_IP/v1/models/sklearn-iris:predict -d @./iris-input2.json
```