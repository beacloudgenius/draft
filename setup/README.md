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

    Make a note of *your* EXTERNAL-IP addresses.

## Set DNS entry

Now that we have an ingress, we need to configure a DNS wildcard record to point to it. You may use any sub-domain name to go with your domain name like I am using `workshop`.

In your preferred DNS tool, you will need to associate:

    *.workshop.[[YOUR-DOMAIN]] --> [[EXTERNAL-IP]]

for example, I am using:

    *.workshop.cloudgeni.us --> 35.197.75.207

verify that with a dig:

    dig xyzrandomwildcard.workshop.cloudgeni.us

    ; <<>> DiG 9.10.6 <<>> xyzrandomwildcard.workshop.cloudgeni.us

    ;; ANSWER SECTION:
    xyzrandomwildcard.workshop.cloudgeni.us. 120	IN	A	35.197.75.207

## Configure Draft

Because we’re using GKE, we can take advantage of GCP’s Container Repository (GCR). You aren’t required to use GCR, but it’s convenient since our Kubernetes cluster is running on GCP too.

### Enable docker to login to gcr

    cat $HOME/.creds/oceanic-isotope-233522-b030dceb4b1a.json | docker login -u _json_key --password-stdin https://gcr.io

Notes: https://cloud.google.com/container-registry/docs/advanced-authentication

and now… finally…

### initialize draft and set registry to point to gcr

    draft init
    draft config set registry gcr.io/oceanic-isotope-233522

side note on packs:

`draft init` installs the built-in Draft packs. _If you are interested in creating your own packs_, you can simply create those packs in your local `$(draft home)/packs` directory.

```
packs/github.com/Azure/draft/packs
  |
  |- PACKNAME
  |     |
  |     |- charts/
  |     |    |- Chart.yaml
  |     |    |- ...
  |     |- Dockerfile
  |     |- detect
  |     |- ...
  |
  |- PACK2
        |-...
```

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

kubectl get pods
draft connect
open localhost:33587

Update the Application

Now, let's change the output in app.py to output "Hello, Cloud Genius!" instead:

    \$ cat <<EOF > app.py
    from flask import Flask

    app = Flask(**name**)

    @app.route("/")
    def hello():
    return "Hello, Cloud Genius!\n"

    if **name** == "**main**":
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

Edit this `cluster-issuer.yaml` file

- use your email address not nilesh@cloudgeni.us

- change to staging TLS to avoid rate limit issue during testing.

  https://acme-staging-v02.api.letsencrypt.org/directory

- and install fake root from https://letsencrypt.org/docs/staging-environment/

## Change app.yaml to request TLS cert

cd example app

- Swap the templates/ingress.yaml with ingress-tls.yaml

- Set basedomain workshop.cloudgeni.us in values.yaml and enable ingress in values.yaml

```yaml
ingress:
  enabled: true
basedomain: workshop.cloudgeni.us
```

or handle ingress via values.yaml like this.

```yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"

  path: /

  hosts:
    - example-erlang.workshop.cloudgeni.us

  tls:
    - secretName: example-erlang.workshop.cloudgeni.us
      hosts:
        - example-erlang.workshop.cloudgeni.us
```

debug helm indentation if needed with a dry run

    helm install --debug --dry-run ./charts/example-erlang

then

draft up
