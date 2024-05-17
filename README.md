# argocd-apps

This repository is an app-of-apps (or in reality an app-of-appsets) which configures Argo CD itself as well as
[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack), which 
configures Prometheus and Grafana.

Argo CD is an `ApplicationSet` which currently only selects the local cluster. This could have been a plain `Application`
but this is a showcase to show how that `ApplicationSet` can easily be extended to deploy to other clusters if 
necessary.

kube-prometheus-stack is an appset which makes use of the [multiple sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/) 
feature. The reason for using this feature is for being able to make use of the upstream Helm chart, and for being able to
override some of the default values. This also makes use of label selectors, where it will match a label `environment`
with the values of `dev`, `staging` or `prod`. When following the installation instructions in [argocd-deploy](https://github.com/blakepettersson/argocd-deploy), 
the `in-cluster` cluster will have its value set to `dev`.

The values repo can be found [here](https://github.com/blakepettersson/argocd-kube-prometheus-stack), which contains
the overridden values. The values are more or less the defaults provided by the upstream Helm chart, except for it
not installing Alertmanager and any rules associated with it, and it also installs the standard Argo CD dashboard. It
also overrides the Grafana and Prometheus services to use `NodePort` instead of `ClusterIP`.

## Promotions between environments

This appset showcases how e.g. version promotion can work, since we can select a distinct values repo ref and Helm chart 
version per environment (it's currently the same version for all envs though).

The key to do promotions between environments in this setup is in this line:

```yaml
chartRevision: '{{ index (dict "dev" "55.1.0" "staging" "55.1.0" "prod" "55.1.0")  (index .metadata.labels "environment")}}'
```

This looks like a bunch of gobbledygook due to Go Templating syntax being what it is, but this is the way to do 
maps/dictionaries in `ApplicationSet`s.

Rewritten into something slightly saner, this is what it would look like if this had a hypothetical Javascript syntax:

```javascript
const metadata = {
    labels: {
        environment: "one of dev/staging/prod"
    }
}
const chartRevision = {
    dev: "55.1.0",
    staging: "55.1.0",
    prod: "55.1.0"
}[metadata.labels.environment]
```

On the other hand we don't actually have to use Go templating syntax - what is shown above is a shorthand for doing this:

```yaml
    - clusters:
        selector:
          matchLabels:
            environment: dev
        values:
          chartRevision: 55.1.0
    - clusters:
        selector:
          matchLabels:
            environment: staging
        values:
          chartRevision: 55.1.0
    - clusters:
        selector:
          matchLabels:
            environment: prod
        values:
          chartRevision: 55.1.0
```

With the cluster generator, all clusters will have a `metadata.labels` dictionary which can be used to determine the 
chart version to install. This setup assumes that there is a label on every cluster called `environment`. This 
parameterizes the version to use within each environment, so that distinct versions can be installed for each environment.

With the current setup, if we would want to promote an artifact we would need to add a commit to the repo for every 
promotion we would like to do, e.g. to do an initial upgrade to `dev` we would change the entry next to `dev`.

```yaml
# Let's update dev!
chartRevision: '{{ index (dict "dev" "55.2.0" "staging" "55.1.0" "prod" "55.1.0")  (index .metadata.labels "environment")}}'
```

Once that has been committed to the Git repository, any cluster that has been labeled with `dev` will update its 
`kube-prometheus-stack` to 55.2.0. Then to upgrade every environment, subsequent commits would be performed to update
`staging` and `prod` to `55.2.0`.

```yaml
# Let's update staging now!
chartRevision: '{{ index (dict "dev" "55.2.0" "staging" "55.2.0" "prod" "55.1.0")  (index .metadata.labels "environment")}}'
```

```yaml
# And then prod!
chartRevision: '{{ index (dict "dev" "55.2.0" "staging" "55.2.0" "prod" "55.2.0")  (index .metadata.labels "environment")}}'
```

This implies 3 separate Git commits to do an upgrade across environments.

We can go deeper into the rabbit hole and make the cluster upgrades fully dynamic, without needing to do separate Git 
commits for promoting artifacts.

> [!WARNING]
> What I am about to show is _completely_ against the principles of Gitops, namely because the state of the versioning is not 
> handled in Git itself.

Instead of parameterizing the chart revision by a clusters environment, let's instead have another label to control the
revision. We can call it `kube-prometheus-stack-version` for clarity. We then simply replace the `chartRevision` line
to instead look like this:

```yaml
# Now the Helm chart version can be dynamically controlled outside Git - this assumes that all clusters have a 
# `kube-prometheus-stack-version` set.
chartRevision: '{{ index .metadata.labels "kube-prometheus-stack-version" }}'
```

How we update the labels to the relevant version is an exercise for the reader, but this can be done with e.g. the 
HTTP API or with the gRPC API, or manually.