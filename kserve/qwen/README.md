kubectl create secret generic hf-secret \
--from-literal=HF_TOKEN=$MY_TOKEN \
-n kserve-test

kubectl apply -f hf-storage.yaml

kubectl apply -f qwen-llm.yaml
