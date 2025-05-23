# Resource Health

## Overview
Argo CD provides built-in health assessment for several standard Kubernetes types, which is then
surfaced to the overall Application health status as a whole. The following checks are made for
specific types of Kubernetes resources:

### Deployment, ReplicaSet, StatefulSet, DaemonSet
* Observed generation is equal to desired generation.
* Number of **updated** replicas equals the number of desired replicas.

### Service
* If service type is of type `LoadBalancer`, the `status.loadBalancer.ingress` list is non-empty,
with at least one value for `hostname` or `IP`.

### Ingress
* The `status.loadBalancer.ingress` list is non-empty, with at least one value for `hostname` or `IP`.

### Job
* If job `.spec.suspended` is set to 'true', then the job and app health will be marked as suspended.
### PersistentVolumeClaim
* The `status.phase` is `Bound`

### Argocd App

The health assessment of `argoproj.io/Application` CRD has been removed in argocd 1.8 (see [#3781](https://github.com/argoproj/argo-cd/issues/3781) for more information).
You might need to restore it if you are using app-of-apps pattern and orchestrating synchronization using sync waves. Add the following resource customization in
`argocd-cm` ConfigMap:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  resource.customizations.health.argoproj.io_Application: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    if obj.status ~= nil then
      if obj.status.health ~= nil then
        hs.status = obj.status.health.status
        if obj.status.health.message ~= nil then
          hs.message = obj.status.health.message
        end
      end
    end
    return hs
```

## Custom Health Checks

Argo CD supports custom health checks written in [Lua](https://www.lua.org/). This is useful if you:

* Are affected by known issues where your `Ingress` or `StatefulSet` resources are stuck in `Progressing` state because of bug in your resource controller.
* Have a custom resource for which Argo CD does not have a built-in health check.

There are two ways to configure a custom health check. The next two sections describe those ways.

### Way 1. Define a Custom Health Check in `argocd-cm` ConfigMap

Custom health checks can be defined in
```yaml
  resource.customizations.health.<group>_<kind>: |
```
field of `argocd-cm`. If you are using argocd-operator, this is overridden by [the argocd-operator resourceCustomizations](https://argocd-operator.readthedocs.io/en/latest/reference/argocd/#resource-customizations).

The following example demonstrates a health check for `cert-manager.io/Certificate`.

```yaml
data:
  resource.customizations.health.cert-manager.io_Certificate: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Ready" and condition.status == "False" then
            hs.status = "Degraded"
            hs.message = condition.message
            return hs
          end
          if condition.type == "Ready" and condition.status == "True" then
            hs.status = "Healthy"
            hs.message = condition.message
            return hs
          end
        end
      end
    end

    hs.status = "Progressing"
    hs.message = "Waiting for certificate"
    return hs
```

In order to prevent duplication of custom health checks for potentially multiple resources, it is also possible to
specify a wildcard in the resource kind, and anywhere in the resource group, like this:

```yaml
  resource.customizations: |
    ec2.aws.crossplane.io/*:
      health.lua: |
        ...
```

```yaml
  # If a key _begins_ with a wildcard, please ensure that the GVK key is quoted.
  resource.customizations: |
    "*.aws.crossplane.io/*":
      health.lua: |
        ...
```

!!!important
    Please, note that wildcards are only supported when using the `resource.customizations` key, the `resource.customizations.health.<group>_<kind>`
    style keys do not work since wildcards (`*`) are not supported in Kubernetes configmap keys.

The `obj` is a global variable which contains the resource. The script must return an object with status and optional message field.
The custom health check might return one of the following health statuses:

  * `Healthy` - the resource is healthy
  * `Progressing` - the resource is not healthy yet but still making progress and might be healthy soon
  * `Degraded` - the resource is degraded
  * `Suspended` - the resource is suspended and waiting for some external event to resume (e.g. suspended CronJob or paused Deployment)

By default, health typically returns a `Progressing` status.

NOTE: As a security measure, access to the standard Lua libraries will be disabled by default. Admins can control access by
setting `resource.customizations.useOpenLibs.<group>_<kind>`. In the following example, standard libraries are enabled for health check of `cert-manager.io/Certificate`.

```yaml
data:
  resource.customizations.useOpenLibs.cert-manager.io_Certificate: true
  resource.customizations.health.cert-manager.io_Certificate: |
    # Lua standard libraries are enabled for this script
```

### Way 2. Contribute a Custom Health Check

A health check can be bundled into Argo CD. Custom health check scripts are located in the `resource_customizations` directory of [https://github.com/argoproj/argo-cd](https://github.com/argoproj/argo-cd). This must have the following directory structure:

```
argo-cd
|-- resource_customizations
|    |-- your.crd.group.io               # CRD group
|    |    |-- MyKind                     # Resource kind
|    |    |    |-- health.lua            # Health check
|    |    |    |-- health_test.yaml      # Test inputs and expected results
|    |    |    +-- testdata              # Directory with test resource YAML definitions
```

Each health check must have tests defined in `health_test.yaml` file. The `health_test.yaml` is a YAML file with the following structure:

```yaml
tests:
- healthStatus:
    status: ExpectedStatus
    message: Expected message
  inputPath: testdata/test-resource-definition.yaml
```

To test the implemented custom health checks, run `go test -v ./util/lua/`.

The [PR#1139](https://github.com/argoproj/argo-cd/pull/1139) is an example of Cert Manager CRDs custom health check.

Please note that bundled health checks with wildcards are not supported.

## Overriding Go-Based Health Checks

Health checks for some resources were [hardcoded as Go code](https://github.com/argoproj/gitops-engine/tree/master/pkg/health) 
because Lua support was introduced later. Also, the logic of health checks for some resources were too complex, so it 
was easier to implement it in Go.

It is possible to override health checks for built-in resource. Argo will prefer the configured health check over the
Go-based built-in check.

The following resources have Go-based health checks:

* PersistentVolumeClaim
* Pod
* Service
* apiregistration.k8s.io/APIService
* apps/DaemonSet
* apps/Deployment
* apps/ReplicaSet
* apps/StatefulSet
* argoproj.io/Workflow
* autoscaling/HorizontalPodAutoscaler
* batch/Job
* extensions/Ingress
* networking.k8s.io/Ingress

## Health Checks

An Argo CD App's health is inferred from the health of its immediate child resources (the resources represented in 
source control). The App health will be the worst health of its immediate child sources. The priority of most to least 
healthy statuses is: `Healthy`, `Suspended`, `Progressing`, `Missing`, `Degraded`, `Unknown`. So, for example, if an App
has a `Missing` resource and a `Degraded` resource, the App's health will be `Missing`.

But the health of a resource is not inherited from child resources - it is calculated using only information about the 
resource itself. A resource's status field may or may not contain information about the health of a child resource, and 
the resource's health check may or may not take that information into account.

The lack of inheritance is by design. A resource's health can't be inferred from its children because the health of a
child resource may not be relevant to the health of the parent resource. For example, a Deployment's health is not
necessarily affected by the health of its Pods. 

```
App (healthy)
└── Deployment (healthy)
    └── ReplicaSet (healthy)
        └── Pod (healthy)
    └── ReplicaSet (unhealthy)
        └── Pod (unhealthy)
```

If you want the health of a child resource to affect the health of its parent, you need to configure the parent's health
check to take the child's health into account. Since only the parent resource's state is available to the health check,
the parent resource's controller needs to make the child resource's health available in the parent resource's status 
field.

```
App (healthy)
└── CustomResource (healthy) <- This resource's health check needs to be fixed to mark the App as unhealthy
    └── CustomChildResource (unhealthy)
```
## Ignoring Child Resource Health Check in Applications

To ignore the health check of an immediate child resource within an Application, set the annotation `argocd.argoproj.io/ignore-healthcheck` to `true`. For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    argocd.argoproj.io/ignore-healthcheck: "true"
```

By doing this, the health status of the Deployment will not affect the health of its parent Application.
