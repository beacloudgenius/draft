## Setup and initialize helm

    wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.0-linux-amd64.tar.gz
    tar xzf helm*gz
    sudo mv linux-amd64/helm /usr/local/bin
    chmod +x /usr/local/bin/helm
    rm helm*gz linux-amd64
   
    kubectl apply -f https://raw.githubusercontent.com/beacloudgenius/k8s-ingress-exercise/master/ingress-nginx/tiller-rbac.yaml

    helm init --service-account tiller --upgrade

## Install ingress

    helm install --name ng stable/nginx-ingress \
        --set rbac.create=true \
        --set controller.image.repository="quay.io/kubernetes-ingress-controller/nginx-ingress-controller" \
        --set controller.image.tag="0.23.0"

    kubectl get svc --watch ng-nginx-ingress-controller

    Make a note of *your* CLUSTER-IP and EXTERNAL-IP addresses.

## Set DNS entry

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

### Enable docker to login to gcr

    cat /home/user/creds/oceanic-isotope-233522-b030dceb4b1a.json | docker login -u _json_key --password-stdin https://gcr.io

Notes: https://cloud.google.com/container-registry/docs/advanced-authentication    

and now… finally…

### initialize draft and set registry to point to gcr

    draft init 
    draft config set registry gcr.io/oceanic-isotope-233522

## Install cert-manager

    helm install \
        --name cm \
        --namespace kube-system \
        --set ingressShim.defaultIssuerName=letsencrypt-prod \
        --set ingressShim.defaultIssuerKind=ClusterIssuer \
        --version v0.5.2 \
        stable/cert-manager


cd example app
draft create
draft up

Swap the templates/ingress.yaml 
set basedomain in values.yaml
and enable ingress in values.yaml

draft up

kubectl get pods
draft connect
open localhost:33587

Update the Application

Now, let's change the output in app.py to output "Hello, Cloud Genius!" instead:

$ cat <<EOF > app.py
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, Cloud Genius!\n"

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8080)
EOF



draft up
draft connect

draft delete

## Enable ingress

Edit values.yaml 

```bash
ingress:
  enabled: true
basedomain: workshop.cloudgeni.us
```

## Enable TLS

    wget https://raw.githubusercontent.com/beacloudgenius/k8s-ingress-exercise/master/ingress-nginx/cluster-issuer.yaml

edit this file and use your email address

    kubectl apply -f cluster-issuer.yaml

annotate values.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: coursebook-ui
  annotations:
    kubernetes.io/ingress.class: nginx
    # Add to generate certificates for this ingress
    kubernetes.io/tls-acme: "true"
spec:
  rules:
    - host: be.a.cloudgeni.us
      http:
        paths:
          - backend:
              serviceName: coursebook-ui
              servicePort: 4004
            path: /
  tls:
    # With this configuration kube-lego will generate a secret in namespace foo called `coursebook-tls`
    # for the URL `coursebook.cloudgeni.us`
    - hosts:
        - "be.a.cloudgeni.us"
      secretName: coursebook-ui-tls
