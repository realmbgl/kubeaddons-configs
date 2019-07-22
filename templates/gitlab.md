# GitLab kubeaddon

This document describes how to install GitLab on Konvoy.
Services are exposed over HTTPS, with certificates obtained using LetsEncrypt.

## Prerequisites

LetsEncrypt requires that GitLab's services be exposed on publicly resolvable hostnames. You will need one DNS A record each for GitLab, the GitLab registry, and Minio. You will also need to choose an issues email for the certificates you will be issuing.

## Install

1. Create `gitlab-rails-secret` according to https://docs.gitlab.com/charts/installation/secrets.html#gitlab-rails-secret.
   * This is a workaround for https://gitlab.com/charts/gitlab/issues/1410.
1. Add entries to your cluster's internal DNS that resolve your GitLab domains to their ingress.
   Run `kubectl edit -n kube-system configmap/coredns` and add this configuration to CoreDNS config:
   ```
       rewrite name <GITLAB DOMAIN> gitlab-nginx-ingress-controller.default.svc.cluster.local
       rewrite name <REGISTRY DOMAIN> gitlab-nginx-ingress-controller.default.svc.cluster.local
       rewrite name <MINIO DOMAIN> gitlab-nginx-ingress-controller.default.svc.cluster.local
   ```
1. Copy the `chartReference.values` section from `gitlab.yaml` to `gitlab-values.yaml`, replacing placeholders with your chosen domains and issuer email.
1. Add the GitLab Helm repo, and install the GitLab Helm chart with your configuration values:
   ```
   helm repo add gitlab https://charts.gitlab.io/
   helm repo update
   helm upgrade --install gitlab gitlab/gitlab -f gitlab-values.yaml
   ```
1. Update your external DNS entries to resolve to the gitlab-nginx-ingress-controller service's external IP(s), once it's available.
   This is necessary in order for certmanager to issue certificates that LetsEncrypt can verify.
   * If using AWS, you can use Route53 to create an A record that is an alias of your ingress's ELB. https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html
     Use the ELB's hostname as the alias target: `kubectl get service gitlab-nginx-ingress-controller --template='{{(index .status.loadBalancer.ingress 0).hostname}}'`
1. Wait for GitLab to finish deploying.
1. Open https://`<GITLAB DOMAIN>` in a browser.
