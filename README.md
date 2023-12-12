# argocd-apps

This repository is an app-of-apps (or in reality an app-of-appsets) which configures Argo CD itself as well as
[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack), which 
configures Prometheus and Grafana.

Argo CD is an `ApplicationSet` which currently only selects the local cluster. This could have been a plain `Application`
but this is a showcase to show how that `ApplicationSet` could easily be extended to deploy to other clusters if 
necessary.

kube-prometheus-stack is an appset which makes use of the [multiple sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/) 
feature. The reason for using this feature is for being able to make use of the upstream Helm chart, and for being able to
override some of the default values. This also makes use of label selectors, where it will match a label `environment`
with the values of `dev`, `staging` or `prod`. When following the installation instructions in [argocd-deploy](https://github.com/blakepettersson/argocd-deploy), 
the `in-cluster` cluster will have its value set to `dev`.

There is only one environment with the single cluster, but the appset showcases how e.g. version promotion can work, 
since we can select a distinct values repo ref and Helm chart version per environment (it's all currently the same version
for all envs though).

The values repo can be found [here](https://github.com/blakepettersson/argocd-kube-prometheus-stack), which contains
the overridden values. The values are more or less the defaults provided by the upstream Helm chart, except for it 
not installing Alertmanager and any rules associated with it, and it also installs the standard Argo CD dashboard. It 
also overrides the Grafana and Prometheus services to use `NodePort` instead of `ClusterIP`.


