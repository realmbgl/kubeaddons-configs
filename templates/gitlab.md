# GitLab on Konvoy

This document describes how to install the GitLab Helm chart on Konvoy, with services exposed over HTTPS, using a self-signed wildcard certificate.

This document was tested on Konvoy 0.6 running on AWS, using version 2.1.1 of the GitLab Helm chart.

## Prerequisites

GitLab's services are exposed via name-based virtual servers.
You will need to choose a domain name for each of GitLab, the GitLab registry, and Minio.
This document will use `gitlab.gitlab.local`, `registry.gitlab.local`, and `minio.gitlab.local`, respectively.
For more on configuring GitLab's domain names, see https://docs.gitlab.com/charts/charts/globals.html#configure-host-settings.

The steps in this document will deploy GitLab with services exposed over HTTPS, using a self-signed wildcard certificate.
You may instead use certmanager and LetsEncrypt to generate signed certificates, or provide your own.
For more on configuring TLS, see https://docs.gitlab.com/charts/installation/tls.html.

## Install

1. Create `gitlab-rails-secret` according to https://docs.gitlab.com/charts/installation/secrets.html#gitlab-rails-secret.
   * This is a workaround for https://gitlab.com/charts/gitlab/issues/1410, which prevents this secret from being automatically created.
1. Configure CoreDNS to resolve your GitLab domains to the Traefik ingress controller's internal IP.
   Run `kubectl edit -n kube-system configmap/coredns` and add this configuration to CoreDNS config:
   ```
       rewrite name gitlab.gitlab.local traefik-kubeaddons.kubeaddons.svc.cluster.local
       rewrite name registry.gitlab.local traefik-kubeaddons.kubeaddons.svc.cluster.local
       rewrite name minio.gitlab.local traefik-kubeaddons.kubeaddons.svc.cluster.local
   ```
1. Add an entry to your local `/etc/hosts` that resolves `gitlab.gitlab.local` to one of the Traefik ingress controller's external IPs:
   ```
   echo -e "\n# GitLab on Konvoy\n$(dig +short $(kubectl --namespace=kubeaddons get service traefik-kubeaddons --template='{{(index .status.loadBalancer.ingress 0).hostname}}') | head -1)\tgitlab.gitlab.local" | sudo tee -a /etc/hosts
   ```
1. Create the file `gitlab-values.yaml` with these contents:
   ```
   global:
     hosts:
       # Our chosen domain that will contain `gitlab`, `registry`, and `minio` subdomains.
       domain: gitlab.local
     ingress:
       # Don't configure certmanager, since we aren't installing it.
       configureCertmanager: false
       annotations:
         # This annotation causes Traefik to create a corresponding frontend for each ingress.
         "kubernetes.io/ingress.class": "traefik"
   gitlab-runner:
     # This secret will contain certificates for the GitLab domains.
     # The GitLab runner will not accept self-signed certificates unless they are included in this secret.
     certsSecretName: gitlab-runner-certs
   certmanager:
     # Don't install certmanager, since we're using a self-signed certificate.
     install: false
   nginx-ingress:
     # Don't install an nginx ingress controller, since we're using Traefik.
     enabled: false
   ```
1. Add the GitLab Helm repo, and install the GitLab Helm chart with your configuration values:
   ```
   helm repo add gitlab https://charts.gitlab.io/
   helm repo update
   helm upgrade --install gitlab gitlab/gitlab -f gitlab-values.yaml
   ```
1. Create a secret named `gitlab-runner-certs` that contains certificates for the GitLab domains.
   This is necessary for the GitLab runner to accept self-signed certificates.
   ```
   kubectl get secret gitlab-wildcard-tls --template='{{ index .data "tls.crt" }}' | base64 -D > gitlab.crt
   kubectl create secret generic gitlab-runner-certs --from-file=gitlab.gitlab.local.crt=gitlab.crt --from-file=registry.gitlab.local.crt=gitlab.crt --from-file=minio.gitlab.local.crt=gitlab.crt
   ```
   **NOTE:** You may skip this step if you are using valid signed certificates.
   If you skip this step, you must deploy the GitLab Helm chart without the `gitlab-runner.certsSecretName` configuration parameter.
1. Wait for GitLab to finish deploying.
1. Open https://gitlab.gitlab.local in a browser.

## Notes

### Admin credentials

The admin username is `root`, and the password is stored in the secret `gitlab-gitlab-initial-root-password`. To retrieve it, run:
```
kubectl get secret gitlab-gitlab-initial-root-password --template='{{ .data.password }}' | base64 -D
```

### Git SSL verification

By default, GitLab CI jobs will verify certificates when checking out a Git project.
Verification will fail if GitLab is using a self-signed certificate.
To disable SSL verification, add this configuration to your project's `.gitlab-ci.yml`:
```
variables:
  GIT_SSL_NO_VERIFY: "1"
```
