cd ~/src/github/hashicorp-demoapp/infrastructure
yard up

cd ~/src/github/hashicorp-demoapp/infrastructure/stack/k8s_config
kubectl apply -f .

