### Install Kserve-LLM

helm install kserve-llmisvc-crd oci://ghcr.io/kserve/charts/kserve-llmisvc-crd \
  --version v0.16.0  \
  --namespace kserve --create-namespace

helm upgrade -i kserve-llmisvc oci://ghcr.io/kserve/charts/kserve-llmisvc-resources \
  --version v0.16.0 \\n  --namespace kserve

kubectl delete configmap inferenceservice-config -n kserve

### Qwen
kubectl create secret generic hf-secret \
--from-literal=HF_TOKEN=$MY_TOKEN \
-n kserve-test

kubectl apply -f hf-storage.yaml

kubectl apply -f qwen-llm.yaml

### Code Rest
rest.http

###
POST http://192.168.0.200/openai/v1/chat/completions
Content-Type: application/json
Host: qwen-llm.kserve-test.didc.local

{
  "model": "qwen",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant that provides clear and concise answers."
    },
    {
      "role": "user",
      "content": "이란과 북한의 국제 관계는 어떠한지?"
    }
  ],
  "max_tokens": 150,
  "temperature": 0.7,
  "stream": false
}

###

POST http://192.168.0.200/v1/models/sklearn-iris:predict
Host: sklearn-iris.kserve-test.didc.local
Content-Type: application/json

{
  "instances": [
    [2.8,  5.8,  2.8,  1.4],
    [1.0,  3.4,  4.5,  1.6]
  ]
}
