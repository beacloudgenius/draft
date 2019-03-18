0. wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.0-linux-amd64.tar.gz
   tar xzf helm*gz
   sudo mv linux-amd64/helm /usr/local/bin
   chmod +x /usr/local/bin/helm
   rm helm*gz linux-amd64
   
1. kubectl apply serviceaccount tiller --namespace kube-system
2. kubectl apply -f role-tiller.yaml
3. kubectl apply -f rolebinding-tiller.yaml
4. kubectl apply -f role-tiller-kube-system.yaml
5. kubectl apply -f rolebinding-tiller-kube-system.yaml
6. helm init --upgrade --service-account=tiller

