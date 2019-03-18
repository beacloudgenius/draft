Install helm

    wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.0-linux-amd64.tar.gz
    tar xzf helm*gz
    sudo mv linux-amd64/helm /usr/local/bin
    chmod +x /usr/local/bin/helm
    rm helm*gz linux-amd64
   
kubectl apply -f https://raw.githubusercontent.com/beacloudgenius/k8s-ingress-exercise/master/ingress-nginx/tiller-rbac.yaml

helm init --service-account tiller --upgrade


helm install --name ng stable/nginx-ingress \
    --set rbac.create=true \
    --set controller.image.repository="quay.io/kubernetes-ingress-controller/nginx-ingress-controller" \
    --set controller.image.tag="0.23.0"

kubectl get svc --watch ng-nginx-ingress-controller

Make a note of *your* CLUSTER-IP and EXTERNAL-IP addresses.

Now that we have an ingress, we need to configure a DNS wildcard record to point to it. You may use any sub-domain name to go with your domain name like I am using `workshop`. 

In your preferred DNS tool, you will need to associate:

    *.workshop.[[YOUR-DOMAIN]] --> [[EXTERNAL-IP]]

for example, I am using:

    *.workshop.cloudgeni.us --> 35.197.75.207

verify that with a dig:

dig you.workshop.cloudgeni.us

; <<>> DiG 9.10.6 <<>> you.workshop.cloudgeni.us

;; ANSWER SECTION:
you.workshop.cloudgeni.us. 120	IN	A	35.197.75.207


## Configure Draft

Because we’re using GKE, we can take advantage of GCP’s Container Repository (GCR). You aren’t required to use GCR, but it’s convenient since our Kubernetes cluster is running on GCP too. 

    cat /home/user/creds/oceanic-isotope-233522-b030dceb4b1a.json | docker login -u _json_key --password-stdin https://gcr.io

and now… finally…

    draft init 
    draft config set registry gcr.io


helm install \
    --name cm \
    --namespace kube-system \
    --set ingressShim.defaultIssuerName=letsencrypt-prod \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --version v0.5.2 \
    stable/cert-manager




https://medium.com/@DazWilkin/azure-draft-on-google-container-engine-d1b25530a313