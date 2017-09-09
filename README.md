# Prometheus Operator
[![Build Status](https://jenkins-monitoring-public.prod.coreos.systems/buildStatus/icon?job=po-tests-master&build=1)](https://jenkins-monitoring-public.prod.coreos.systems/job/po-tests-master/1/)
[![Go Report Card](https://goreportcard.com/badge/coreos/prometheus-operator "Go Report Card")](https://goreportcard.com/report/coreos/prometheus-operator)

**Project status: *alpha*** Not all planned features are completed. The API, spec, status 
and other user facing objects are subject to change. We do not support backward-compatibility
for the alpha releases.

The Prometheus Operator for Kubernetes provides easy monitoring definitions for Kubernetes
services and deployment and management of Prometheus instances.

Once installed, the Prometheus Operator provides the following features:

* **Create/Destroy**: Easily launch a Prometheus instance for your Kubernetes namespace,
  a specific application or team easily using the Operator.

* **Simple Configuration**: Configure the fundamentals of Prometheus like versions, persistence, 
  retention policies, and replicas from a native Kubernetes resource.

* **Target Services via Labels**: Automatically generate monitoring target configurations based
  on familiar Kubernetes label queries; no need to learn a Prometheus specific configuration language.

For an introduction to the Prometheus Operator, see the initial [blog
post](https://coreos.com/blog/the-prometheus-operator.html).

**Documentation is hosted on [coreos.com](https://coreos.com/operators/prometheus/docs/latest/)**

The current project roadmap [can be found here](./ROADMAP.md).

## Prometheus Operator vs. kube-prometheus

The Prometheus Operator makes the Prometheus configuration Kubernetes native
and manages and operates Prometheus and Alertmanager clusters. It is a piece of
the puzzle regarding full end-to-end monitoring.

[kube-prometheus](contrib/kube-prometheus) combines the Prometheus Operator
with a collection of manifests to help getting started with monitoring
Kubernetes itself and applications running on top of it.

## Prerequisites

Version `>=0.2.0` of the Prometheus Operator requires a Kubernetes
cluster of version `>=1.5.0`. If you are just starting out with the 
Prometheus Operator, it is highly recommended to use the latest version.

If you have previously used pre-1.5.0 releases of Kubernetes with the `0.1.0` 
version of the Prometheus Operator, see the [migration](#migration) section.

## Migration

The `PetSet` was deprecated in the `1.5.0` release of Kubernetes in favor of
the `StatefulSet`. As the Prometheus Operator used the `PetSet` in version
`0.1.0`, those need to be migrated as we upgrade our Kubernetes cluster as well
as the Prometheus Operator.

First the Prometheus Operator needs to be shut down. Once shut down, retrieve 
the `PetSet`s that were generated by it. You can do so simply by finding all
`Prometheus` and `Alertmanager` objects created:

```
kubectl get prometheuses --all-namespaces
kubectl get alertmanagers --all-namespaces
```

For each `Prometheus` and `Alertmanager` object, a respective `PetSet` with the
same name was created in the same namespace. Those `PetSet`s need to be
migrated according to the [official migration documentation](http://kubernetes.io/docs/tasks/manage-stateful-set/upgrade-pet-set-to-stateful-set/).

Once migrated and on Kubernetes version `>=1.5.0`, you can start the
Prometheus Operator of version `>=0.2.0`, and the `StatefulSet` created
in the migration will from now on be managed by the Prometheus Operator.

## CustomResourceDefinitions

The Operator acts on the following [custom resource definitions (CRDs)](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/):

* **`Prometheus`**, which defines a desired Prometheus deployment.
  The Operator ensures at all times that a deployment matching the resource definition is running.

* **`ServiceMonitor`**, which declaratively specifies how groups
  of services should be monitored. The Operator automatically generates Prometheus scrape configuration
  based on the definition.

* **`Alertmanager`**, which defines a desired Alertmanager deployment.
  The Operator ensures at all times that a deployment matching the resource definition is running.

To learn more about the CRDs introduced by the Prometheus Operator have a look
at the [design doc](Documentation/design.md).

## Installation

Install the Operator inside a cluster by running the following command:

```
kubectl apply -f bundle.yaml
```

> Note: make sure to adapt the namespace in the ClusterRoleBinding if deploying in another namespace than the default namespace.

To run the Operator outside of a cluster:

```
make
hack/run-external.sh <kubectl cluster name>
```

## Removal

To remove the operator and Prometheus, first delete any custom resources you created in each namespace. The
operator will automatically shut down and remove Prometheus and Alertmanager pods, and associated configmaps.

```
for n in $(kubectl get namespaces -o jsonpath={..metadata.name}); do
  kubectl delete --all --namespace=$n prometheus,servicemonitor,alertmanager
done
```

After a couple of minutes you can go ahead and remove the operator itself.

```
kubectl delete -f bundle.yaml
```

The operator automatically creates services in each namespace where you created a Prometheus or Alertmanager resources,
and defines three custom resource definitions. You can clean these up now.

```
for n in $(kubectl get namespaces -o jsonpath={..metadata.name}); do
  kubectl delete --ignore-not-found --namespace=$n service prometheus-operated alertmanager-operated
done

kubectl delete --ignore-not-found customresourcedefinitions \
  prometheus.monitoring.coreos.com \
  service-monitor.monitoring.coreos.com \
  alertmanager.monitoring.coreos.com
```

**The Prometheus Operator collects anonymous usage statistics to help us learning how the software is being used and how we can improve it. To disable collection, run the Operator with the flag `-analytics=false`**

## Development

### Prerequisites

- golang environment
- docker (used for creating container images, etc.)
- minikube (optional)

### Testing

1. Ensure that you're running tests in the following path: `$GOPATH/src/github.com/coreos/prometheus-operator` as tests expect paths to match.
  1. If you're working from a fork, just add the forked repo as a remote and pull against your local coreos checkout before running tests.
1. `make test` executes all *unit tests*.
2. You can execute the *e2e tests* on a local minikube by compiling the static binary (which is what is used for the container images) with `make crossbuild`.
  1. build the container image with the docker host from within minikube by running `eval $(minikube docker-env)`.
  2. You can build the container using `make container`.
  3. Finally run the e2e tests using `make e2e-tests`.
