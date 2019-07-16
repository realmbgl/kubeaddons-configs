# GitLab kubeaddon

The GitLab kubeaddon config deploys GitLab with a few pieces missing. To finish the deployment, follow these steps:

1. Create `gitlab-rails-secret` according to https://docs.gitlab.com/charts/installation/secrets.html#gitlab-rails-secret.
  * This is a workaround for https://gitlab.com/charts/gitlab/issues/1410.
1. Add this configuration to the `Corefile` in the `coredns` ConfigMap under the `kube-system` namespace:
```
    rewrite name gitlab.gitlab.local gitlab-unicorn.default.svc.cluster.local
    rewrite name registry.gitlab.local gitlab-registry.default.svc.cluster.local
    rewrite name minio.gitlab.local gitlab-minio-svc.default.svc.cluster.local
```
1. Wait for GitLab to finish deploying now that it has everything it needs.
1. Add an entry to `/etc/hosts` that resolves `gitlab.gitlab.local` to an IP address that the GitLab service is exposed on:
```
echo -e "\n# GitLab on Konvoy\n$(dig +short $(kubectl get service gitlab-nginx-ingress-controller --template='{{(index .status.loadBalancer.ingress 0).hostname}}') | head -1)\tgitlab.gitlab.local" | sudo tee -a /etc/hosts
```
1. Open https://gitlab.gitlab.local in a browser.

# Notes

  * The GitLab Runner must be disabled when using wildcard self-signed certificates. This kubeaddon config deploys GitLab with wildcard self-signed certificates, so the Runner is disabled.
