# Add the repository (example for nicholaswilde's chart)
helm repo add nicholaswilde https://nicholaswilde.github.io

# Update your local Helm chart repository cache
helm repo update

# Install the code-server chart
helm upgrade code nicholaswilde/code-server --version 1.1.1

kubectl create secret tls code-ingress --key code.key --cert code.crt -n code

