#### Install Rancher

# cert-manager 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml

# Rancher helm 레파지토리 등록
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest 

# Rancher 설치
# 원하는 도메인 명 등록 후 해당 도메인 명으로 접속
helm upgrade -i rancher rancher-latest/rancher \
  --set hostname=rancher.local --set bootstrapPassword=admin \
  --set replicas=1 --set global.cattle.psp.enabled=false \
  --create-namespace -n cattle-system


#### Install metallab

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml 

kubectl apply -f metallb-config.yaml

#### Install nfs-driver

curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.5.0/deploy/install-driver.sh | bash -s v4.5.0 --

kubectl apply -f nfs-sc.yaml

#### Install harbor

kubectl create ns harbor
kubectl create secret tls harbor-ingress --key harbor.key --cert harbor.crt -n harbor

#### Ubuntu
cp harbor.crt harbor.key /usr/local/share/ca-certificates/ 
update-ca-certificates # ubuntu


#### Gateway API CRD

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

#### Istio

helm repo add istio https://istio-release.storage.googleapis.com/charts --force-update

helm install istio-base istio/base -n istio-system --wait --set defaultRevision=default --create-namespace --version 1.27.1

helm install istiod istio/istiod -n istio-system --wait --version 1.27.1 \
   --set proxy.autoInject=disabled \
   --set-string pilot.podAnnotations."cluster-autoscaler\.kubernetes\.io/safe-to-evict"=true

helm install istio-ingressgateway istio/gateway -n istio-system --version 1.27.1 \
   --set-string podAnnotations."cluster-autoscaler\.kubernetes\.io/safe-to-evict"=true

#### Knative-Serviing

helm install knative-operator --namespace knative-serving --create-namespace --wait \
      https://github.com/knative/operator/releases/download/knative-v1.15.7/knative-operator-v1.15.7.tgz

#### Knative-Serving Service

kubectl apply -f kserving.yaml

#### Install KServ

helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd --version v0.16.0 --namespace kserve --create-namespace --wait

helm install kserve oci://ghcr.io/kserve/charts/kserve --version v0.16.0 --namespace kserve --create-namespace --wait --set-string kserve.controller.deploymentMode="Serverless"

kubectl get pods -n kserve

#### Install KServ Service
kubectl create ns kserv-test

kubectl apply -f serving-runtime.yaml

kubectl get servingruntime.serving.kserve.io -n kserv-test

kubectl apply -f inference-service.yaml

kubectl get inferenceservices sklearn-iris -n kserve-test

#### Kserv Iris test
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

curl -v -H "Host: sklearn-iris.kserve-test.didc.local" -H "Content-Type: application/json" \
  http://%LB_IP%/v1/models/sklearn-iris:predict -d @./iris-input.json

curl -v -H "Host: sklearn-iris.kserve-test.didc.local" -H "Content-Type: application/json" \
  http://%LB_IP%/v1/models/sklearn-iris:predict -d @./iris-input2.json
