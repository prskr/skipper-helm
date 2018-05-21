# Helm chart for [Skipper](https://github.com/zalando/skipper)

[![Build Status](https://travis-ci.org/baez90/skipper-helm.svg?branch=master)](https://travis-ci.org/baez90/skipper-helm)

- [Helm chart for Skipper](#helm-chart-for-skipperhttps---githubcom-zalando-skipper)
    - [Helm registry](#helm-registry)
    - [Deployment](#deployment)
        - [Minimal](#minimal)
        - [Other namespace than `default`](#other-namespace-than-default)
        - [Enable RBAC](#enable-rbachttps---kubernetesio-docs-admin-authorization-rbac)
        - [Enable prometheus-operator](#enable-prometheus-operatorhttps---githubcom-coreos-prometheus-operator)
        - [Debugging](#debugging)
        - [Enable `ingress.class` annotation handling](#enable-ingressclass-annotation-handling)
        - [Deploy with `values.yaml` file](#deploy-with-valuesyaml-file)
        - [Running multiple instances](#running-multiple-instances)

## Helm registry

The chart is available at the [Quay.io registry](https://quay.io/application/baez/skipper).

To be able to install the chart you will need the [registry plugin](https://github.com/app-registry/appr-helm-plugin).
Please follow the install guide in the GitHub repository.

## Deployment

### Minimal

The minimal deployment of this chart looks like this:

- install the Helm client
- install the Helm registry plugin

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    "<you release name e.g. skipper>"
```

### Other namespace than `default`

To deploy the ingress controller to a specific namespace run it like this and adjust the **--namespace** value:

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --namespace "<your-namespace>"
    "<you release name e.g. skipper>"
```

### Enable [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/)

Role-Based Access Control (“RBAC”) is stable since Kubernetes 1.8 and is part of the Kubernetes best practices.
This Helm chart includes manifests for all required resources but does **not** deploy them by default.
If you have RBAC enabled in your Kubernetes cluster you need the following additional resources deployed:

- ClusterRole
- ClusterRoleBinding
- ServiceAccount

This is done by passing `--set rbac.create=true` to the `helm` CLI like this:

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set rbac.create=true
    "<you release name e.g. skipper>"
```

There are additional values you can override if you want to customize e.g. the name of the `ServiceAccount`.
The following variables can be overridden:

| Variable                      | Default value |
|-------------------------------|---------------|
| `rbac.svcAccountName`         | skipper       |
| `rbac.clusterRoleName`        | skipper       |
| `rbac.clusterRoleBindingName` | skipper       |

_Note: the `ServiceAccount` will be created in the same namespace as the rest of the chart._

### Enable [prometheus-operator](https://github.com/coreos/prometheus-operator)

Prometheus-Operator is project that deploys [Prometheus](https://prometheus.io/) to your Kubernetes cluster.

_Note: there's also a [Helm chart](https://github.com/coreos/prometheus-operator/tree/master/helm) available for Prometheus-Operator._

To notify Prometheus-Operator that it should collect the metrics of Skipper the following additional resources are required:

- ServiceMonitor

This Helm chart includes the required manifest but does **not** deploy it by default.
To enable support for Prometheus-Operator add the flat `--set prometheusOperator.create=true` to your `helm` CLI like this:

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set prometheusOperator.create=true \
    "<you release name e.g. skipper>"
```

There are a few more configuration options available you can pass to the CLI if required:

| Variable                            | Default value               | Description                                                                                                         |
|-------------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| `prometheusOperator.jobLabel`       | skipper-ingress             | Label of the Prometheus job                                                                                         |
| `prometheusOperator.monitorName`    | skipper-metrics             | Name of the `ServiceMonitor` resource                                                                               |
| `prometheusOperator.namespace`      | monitoring                  | Namespace where your Prometheus-Operator is deployed                                                                |
| `prometheusOperator.scrapeInterval` | 30s                         | Interval how often Prometheus will collect metrics                                                                  |
| `prometheusOperator.labels[]`       | prometheus: kube-prometheus | Set of labels to add to the `ServiceMonitor` the default value reflects the default selector or Prometheus-Operator |

### Debugging

Sometimes something is just going wrong and you have no clue what's happening.
You can set the log level to `DEBUG` to get more insights by adding the flat `--set skipper.logLevel="DEBUG"` to the `helm` CLI like this:

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set skipper.logLevel="DEBUG" \
    "<you release name e.g. skipper>"
```

The available log levels are:

- `ERROR`
- `WARN`
- `INFO`
- `DEBUG`

### Enable `ingress.class` annotation handling

If you want to split the traffic to different Skipper deployments you can define the value `skipper.ingressClass` which will enable the built-in support of Skipper to parse the pod annotation `kubernetes.io/ingress.class`.

Pass the flag `--set skipper.ingressClass=skipper-prod` to the `helm` CLI like this

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set skipper.ingressClass="skipper-prod" \
    "<you release name e.g. skipper>"
```

To enable the parsing support and force Skipper to filter for pods with the annotation:

```yaml
kubernetes.io/ingress.class: skipper-prod
```

### Deploy with `values.yaml` file

If you don't want to pass all options via `--set` you can also copy the shipped `./skipper/values.yaml`, adopt it and pass it to the `helm` CLI like this:

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    -f my-values.yaml \
    "<you release name e.g. skipper>"
```

### Running multiple instances

If you want to run multiple instances of Skipper and the ingress controller make sure that you change some default values to avoid naming and port clashes.

A bare minimal scenario might look like this to avoid port clashes:

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set skipper.ingressClass="skipper-prod" \
    --set skipper.webEndpoint.port=9999 \
    --set skipper.metricsEndpoint.port=9911 \
    --set skipper.
  "<you release name e.g. skipper-prod>"

helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set skipper.ingressClass="skipper-staging" \
    --set skipper.webEndpoint.port=9988 \
    --set skipper.metricsEndpoint.port=9922 \
  "<you release name e.g. skipper-staging>"
```

Things are getting a little bit more complicated if it comes to RBAC and Prometheus-Operator:

```bash
helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set skipper.ingressClass="skipper-prod" \
    --set skipper.webEndpoint.port=9999 \
    --set skipper.metricsEndpoint.port=9911 \
    --set rbac.create=true \
    --set rbac.svcAccountName="skipper-prod" \
    --set rbac.clusterRoleName="skipper-prod" \
    --set rbac.clusterRoleBindingName="skipper-prod" \
    --set skipper.
  "<you release name e.g. skipper-prod>"

helm registry upgrade quay.io/baez/skipper -- \
    --install \
    --wait \
    --set skipper.ingressClass="skipper-staging" \
    --set skipper.webEndpoint.port=9988 \
    --set skipper.metricsEndpoint.port=9922 \
    --set rbac.create=true \
    --set rbac.svcAccountName="skipper-staging" \
    --set rbac.clusterRoleName="skipper-staging" \
    --set rbac.clusterRoleBindingName="skipper-staging" \
  "<you release name e.g. skipper-staging>"
```