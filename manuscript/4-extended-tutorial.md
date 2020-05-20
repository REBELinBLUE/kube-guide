# Extended Tutorial

Now that you have followed through a step by step tutorial, we will discuss some more advanced features. In this tutorial we will cover the following topics

- Editing and Patching Resources
- Using *HorizontalPodAutoscalers*
- Applying *ResourceQuota* and *LimitRange*
- Using *NetworkPolicy*
- Using *PodSecurityPolicy* and *PodDisruptionBudget*
- Using multiple clusters
- *ServiceAccounts*, *Roles* and *RoleBindings*

## Editing and Patching Resources

Until now we have updated resources by changing our manifest files and using `kubectl apply` to apply the new version, but there are a couple of other ways to update resources.

### Editing Resources

The first way is using the `edit` command, you can use this to edit a specific resource or all resources of a type; when you run the command the file will be opened in the editor defined by your shell's `$KUBE_EDITOR` or `$EDITOR` variables (normally `vi` if not set).

Let's try to edit the kuard *Deployment*.

```bash
❯ kubectl edit deployment/kuard
```

When the file opens you will notice there are a lot more fields than we originally created the *Deployment* with. The `status` field is managed by *Kubelet* and can't be edited, other fields are default values. Kubernetes has a concept of [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) which run when you submit a resource to the API, prior to the resource being created. There are several built in controllers but you are also able to define your own. There are 2 types of *Admission Controllers*, the first *MutatingAdmissionWebhook* makes changes to the resource such as sets defaults, the second *ValidatingAdmissionWebhook* checks the resource and either accepts or rejects it.

For example, in the previous tutorial we mentioned how *Pods* have default *tolerations* for *Node* health, these are set by the `DefaultTolerationSeconds` *Admission Controller*.

Change the image tag in the editor from `purple` to `green` and then exit the editor, the resource will be changed.

```bash
❯ kubectl edit deployment/kuard
deployment.apps/kuard edited
```

If you make a mistake Kubernetes will reject the changes

```bash
❯ kubectl edit deployment/kuard
error: deployments.apps "kuard" is invalid
```

Some fields are immutable, if you try to change to change these then Kubernetes will also reject the changes

```bash
❯ kubectl edit deployment/kuard
error: At least one of apiVersion, kind and name was changed
```

This is all well and good for a real person, although it is less than ideal (really you should keep your manifests under version control and apply the changed version), but what if you want to change a resource programmatically, for example as part of a CI system?

### Patching Resources

There are 2 commands available

`kubectl set` for setting a few specific fields on the *Pod* template (env, image, resources) and other fields on a few other resources.

```bash
❯ kubectl set --help
Configure application resources

 These commands help you make changes to existing application resources.

Available Commands:
  env            Update environment variables on a pod template
  image          Update image of a pod template
  resources      Update resource requests/limits on objects with pod templates
  selector       Set the selector on a resource
  serviceaccount Update ServiceAccount of a resource
  subject        Update User, Group or ServiceAccount in a RoleBinding/ClusterRoleBinding

Usage:
  kubectl set SUBCOMMAND [options]
```

For example, you would update the image using the following command. This updates the kuard container of the *Pod* template defined in the *Deployment* kuard.

```bash
❯ kubectl set image deployment/kuard kuard=gcr.io/kuar-demo/kuard-amd64:blue
deployment.apps/kuard image updated
```

More useful though is `kubectl patch`, this allows you to patch any resource using YAML or JSON. 

Create a file `14-patch.yaml` with the following content

```yaml
spec:
  template:
    spec:
      containers:
      - name: kuard
        image: gcr.io/kuar-demo/kuard-amd64:purple
```

This patch file tells Kubernetes which part of the manifest to update. You may notice that the `name` of the container is supplied even though it is not being changed, if it is not supplied Kubernetes would not know whether we are trying to update a container or add a new one to the *Deployment*.

If we did not supply the name we receive an error

```bash
Error from server: map: map[image:gcr.io/kuar-demo/kuard-amd64:purple] does not contain declared merge key: name
```

This is known as a "Strategic Merge" because Kubernetes will attempt to decide what to do based on the fields in the patch file. If we had multiple containers defined in this patch the new ones will be added, but if we have multiple defined in the *Deployment*, the ones missing from the patch will not be removed.

Kubernetes supports 2 other merge strategies, "JSON Merge Patch" based on [RFC 7386](https://tools.ietf.org/html/rfc7386), using this strategy the entire container list will be replaced; and "JSON Patch" based on [RFC 6902](https://tools.ietf.org/html/rfc6902) which lets you specify whether individual paths are added, removed or replaced. You can read more about these strategies in the [Kubernetes documentation](https://kubernetes.io/docs/tasks/run-application/update-api-object-kubectl-patch/).

Now apply the patch

```bash
❯ kubectl patch deployment kuard --patch "$(cat 14-patch.yaml)"
deployment.apps/kuard patched
```

If you `describe` the *Deployment* you will see that the image has been changed, and of course a new *Pod* will be created.

```bash
❯ kubectl describe deployment kuard
Name:                   kuard
...
Pod Template:
  Labels:  app=kuard
  Containers:
   kuard:
    Image:        gcr.io/kuar-demo/kuard-amd64:purple
...
```

Now change the patch, modify the name of the container to `second-container` and apply it again.

This time if you `describe` the *Deployment* you will see that it now has 2 containers. This is because the "Strategic Merge" strategy sees that no container exists with the name supplied, so it adds one.

```bash
❯ kubectl describe deployment kuard
Name:                   kuard
...
Pod Template:
  Labels:  app=kuard
  Containers:
   second-container:
    Image:        gcr.io/kuar-demo/kuard-amd64:purple
    ...
   kuard:
    Image:        gcr.io/kuar-demo/kuard-amd64:purple
...
```

Next change the image tag to `green` then apply the file and `describe` the *Deployment* again. This time you should see that the new container has had it's image changed, but the original container is left unchanged

```bash
❯ kubectl describe deployment kuard
Name:                   kuard
...
Pod Template:
  Labels:  app=kuard
  Containers:
   second-container:
    Image:        gcr.io/kuar-demo/kuard-amd64:green
    ...
   kuard:
    Image:        gcr.io/kuar-demo/kuard-amd64:purple
...
```

Finally, reset the container name in the patch to `kuard` and then let's apply it using the "JSON Merge Patch" strategy

```bash
❯ kubectl patch deployment kuard --type merge --patch "$(cat manuscript/resources/code/14-patch.yaml)"
deployment.apps/kuard patched
```

This time if you `describe` the *Deployment* you will see that it is back to only having the one container, with the new image tag, because the "JSON Merge Patch" strategy replaces the entire container list.

So far we have just used a static file to patch the deployment, we could generate this programmatically but it is a bit of a faff. Instead we can use the [jq](https://stedolan.github.io/jq/) command line tool to create valid JSON data for us which can be used to patch.

```bash
❯ jq -c -n --arg name kuard --arg image gcr.io/kuar-demo/kuard-amd64:blue '{ spec: { template: { spec: { containers:[ { name: $name, image: $image } ] } } } }'

{"spec":{"template":{"spec":{"containers":[{"name":"kuard","image":"gcr.io/kuar-demo/kuard-amd64:blue"}]}}}}
```

If we were to store at template for our patch in a file, for example `patch.jq`

```jq
{
  spec: {
    template: {
      spec: {
        containers: [
          {
            name: $name,
            image: $image
          }
        ]
      }
    }
  }
}
```

We can then generate the patch

```bash
❯ jq -c -n --arg name kuard --arg image gcr.io/kuar-demo/kuard-amd64:blue -f patch.jq

{"spec":{"template":{"spec":{"containers":[{"name":"kuard","image":"gcr.io/kuar-demo/kuard-amd64:blue"}]}}}}
```

Hopefully you can see how we can use `jq` to generate patches for us without having to worry about doing things like escaping values ourselves, `jq` has lots of powerful features for manipulating JSON.

So we can now patch our *Deployment* using the following

```bash
❯ JSON_PATCH=$(jq -c -n --arg name kuard --arg image gcr.io/kuar-demo/kuard-amd64:blue -f patch.jq)

❯ kubectl patch deployment kuard --patch "$JSON_PATCH"
deployment.apps/kuard patched
```

## Using HorizontalPodAutoscalers

*HorizontalPodAutoscalers* allow you to configure your cluster to automatically add and remove *Pods* based on resource usage. The current stable version of the *HPA* API (autoscaling/v1) only supports scaling based upon CPU usage, however the current beta (autoscaling/v2beta2) also supports scaling based on memory and on custom metrics exposed by a metrics server.

You can see the difference using the `kubectl explain` command.

```bash
❯ kubectl explain horizontalpodautoscaler.spec
KIND:     HorizontalPodAutoscaler
VERSION:  autoscaling/v1

RESOURCE: spec <Object>

DESCRIPTION:
     behaviour of autoscaler. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status.

     specification of a horizontal pod autoscaler.

FIELDS:
   maxReplicas	<integer> -required-
     upper limit for the number of pods that can be set by the autoscaler;
     cannot be smaller than MinReplicas.

   minReplicas	<integer>
     minReplicas is the lower limit for the number of replicas to which the
     autoscaler can scale down. It defaults to 1 pod. minReplicas is allowed to
     be 0 if the alpha feature gate HPAScaleToZero is enabled and at least one
     Object or External metric is configured. Scaling is active as long as at
     least one metric value is available.

   scaleTargetRef	<Object> -required-
     reference to scaled resource; horizontal pod autoscaler will learn the
     current resource consumption and will set the desired number of pods by
     using its Scale subresource.

   targetCPUUtilizationPercentage	<integer>
     target average CPU utilization (represented as a percentage of requested
     CPU) over all the pods; if not specified the default autoscaling policy
     will be used.
```

Add the `--api-version` parameter to get the details for the specific version of the API

```bash
❯ kubectl explain --api-version=autoscaling/v2beta2 horizontalpodautoscaler.spec
KIND:     HorizontalPodAutoscaler
VERSION:  autoscaling/v2beta2

RESOURCE: spec <Object>

DESCRIPTION:
     spec is the specification for the behaviour of the autoscaler. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status.

     HorizontalPodAutoscalerSpec describes the desired functionality of the
     HorizontalPodAutoscaler.

FIELDS:
   behavior	<Object>
     behavior configures the scaling behavior of the target in both Up and Down
     directions (scaleUp and scaleDown fields respectively). If not set, the
     default HPAScalingRules for scale up and scale down are used.

   maxReplicas	<integer> -required-
     maxReplicas is the upper limit for the number of replicas to which the
     autoscaler can scale up. It cannot be less that minReplicas.

   metrics	<[]Object>
     metrics contains the specifications for which to use to calculate the
     desired replica count (the maximum replica count across all metrics will be
     used). The desired replica count is calculated multiplying the ratio
     between the target value and the current value by the current number of
     pods. Ergo, metrics used must decrease as the pod count is increased, and
     vice-versa. See the individual metric source types for more information
     about how each type of metric must respond. If not set, the default metric
     will be set to 80% average CPU utilization.

   minReplicas	<integer>
     minReplicas is the lower limit for the number of replicas to which the
     autoscaler can scale down. It defaults to 1 pod. minReplicas is allowed to
     be 0 if the alpha feature gate HPAScaleToZero is enabled and at least one
     Object or External metric is configured. Scaling is active as long as at
     least one metric value is available.

   scaleTargetRef	<Object> -required-
     scaleTargetRef points to the target resource to scale, and is used to the
     pods for which metrics should be collected, as well as to actually change
     the replica count.
```

You will see that `targetCPUUtilizationPercentage` has been replaced with `metrics`.

Previously we created a *Pod* designed to increase memory usage until it was `OOMKilled`, now let's create a *Deployment* so that we can experiment with *HorizontalPodAutoscalers*. 

Create the file `15-deployment-with-memory.yaml` and apply it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-test
spec:
  selector:
    matchLabels:
      app: hpa
  template:
    metadata:
      labels:
        app: hpa
    spec:
      containers:
        - name: stress
          image: polinux/stress
          resources:
            requests:
              memory: 50Mi
              cpu: 100m
            limits:
              memory: 75Mi
              cpu: 110m
          command: ["stress"]
          args: ["--vm", "11", "--vm-bytes", "5M", "--vm-hang", "1", "--vm-keep"]
```

If you then get the *Pod* resource using `kubectl top pods` you will see that the *Pod* is using quite a lot of memory (note it may take a few minutes before the metrics start appearing).

```bash
❯ kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)
hpa-test-565868f8b4-2tt4r   0m           56Mi
```

Now let's create a *HorizontalPodAutoscaler* to add more *Pods* when the memory usage is over 85%, this is based on the `requests` rather than the `limits`, so in this example Kubernetes will add new *Pods* when the 5 minute average memory usage is above 42.5Mi and it will keep adding *Pods* until the memory usage is back under the target. Kubernetes will also remove *Pods* when they are no longer needed.

> WARNING: Since we are using a stress testing tool in this example the memory usage will never drop so make sure you set `maxReplicas` to a fairly small value, otherwise Kubernetes will keep adding *Pods* and could end up filling your *Nodes* completely.

Create a file with the following content.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-test
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-test
  maxReplicas: 5
  minReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        averageUtilization: 70
        type: Utilization
```

The `spec.scaleTargetRef` field tells the *HorizontalPodAutoscaler* which object to target, either a *Deployment*, *ReplicaSet* or *StatefulSet*. You may be wondering why not *DaemonSet*, but if you think back, a *DaemonSet* is used to ensure a single *Pod* for the set is run on every single *Node*. 

Apply the file and then get the *HorizontalPodAutoscaler*

```bash
❯ kubectl get horizontalpodautoscaler/hpa-test
NAME       REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-test   Deployment/hpa-test   <unknown>/70%   1         5         0          4s
```

You will notice that the TARGETS is currently `<unknown>`, this is because it has not yet been running for long enough to work out a running average. Try again and few times and you should then get a value.

```bash
❯ kubectl get horizontalpodautoscaler/hpa-test
NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
hpa-test   Deployment/hpa-test   112%/70%   1         5         2          32s
```

As you can see here, the average is currently at 112%, so the HPA has added a second replica. If you watch the *Pods* you will notice it will continue adding *Pods* until there are 5 (the `maxReplicas` define in the *HorizontalPodAutoscaler*).

You can `describe` the *HorizontalPodAutoscaler* and you will see the events which have been triggered, scaling more *Pods* to be added.

```bash
❯ kubectl describe horizontalpodautoscaler/hpa-test
Name:                                                     hpa-test
Namespace:                                                default
Labels:                                                   <none>
Annotations:                                              CreationTimestamp:  Tue, 19 May 2020 23:05:48 +0100
Reference:                                                Deployment/hpa-test
Metrics:                                                  ( current / target )
  resource memory on pods  (as a percentage of request):  112% (58948812800m) / 70%
Min replicas:                                             1
Max replicas:                                             5
Deployment pods:                                          5 current / 5 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from memory resource utilization (percentage of request)
  ScalingLimited  True    TooManyReplicas   the desired replica count is more than the maximum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  2m44s  horizontal-pod-autoscaler  New size: 2; reason: memory resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  2m14s  horizontal-pod-autoscaler  New size: 4; reason: memory resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  73s    horizontal-pod-autoscaler  New size: 5; reason: memory resource utilization (percentage of request) above target
```

If you edit the *HorizontalPodAutoscaler* using the `kubectl edit` command you will notice that the `specs.metrics` field is missing, this is because by default `edit` will use the current stable API. You can specify which version of the API to use in the resource name (although of course normally you won't be editing the resource directly).

```bash
❯ kubectl edit horizontalpodautoscaler.v2beta2.autoscaling/hpa-test
```

Change the `averageUtilization` to something high, say 500 and then close the editor. Get the *HorizontalPodAutoscaler* and you should then see the TARGET is your new value.

If you wait a few minutes you will see the *Pods* start getting removed, this will take a while as Kubernetes uses a 5 minute average.

Again, if you `describe` the *HorizontalPodAutoscaler* you will see in the event list that the metrics are below target.

```bash
❯ kubectl describe horizontalpodautoscaler/hpa-test
Name:                                                     hpa-test
Namespace:                                                default
Labels:                                                   <none>
Annotations:                                              CreationTimestamp:  Tue, 19 May 2020 23:05:48 +0100
Reference:                                                Deployment/hpa-test
Metrics:                                                  ( current / target )
  resource memory on pods  (as a percentage of request):  112% (58943078400m) / 500%
Min replicas:                                             1
Max replicas:                                             5
Deployment pods:                                          2 current / 2 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from memory resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange   the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  13m   horizontal-pod-autoscaler  New size: 2; reason: memory resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  13m   horizontal-pod-autoscaler  New size: 4; reason: memory resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  12m   horizontal-pod-autoscaler  New size: 5; reason: memory resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  23s   horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
```

You can now delete both the *Deployment* and the *HorizontalPodAutoscaler*.

```bash
❯ kubectl delete deployment/hpa-test horizontalpodautoscaler/hpa-test
deployment.apps "hpa-test" deleted
horizontalpodautoscaler.autoscaling "hpa-test" deleted
```

It is recommended that you at least have *HorizontalPodAutoscalers* setup based on `cpu` to prevent throttling, you can also set them up based on `memory` as we did here, however many applications do not clean up their memory usage aggressively so you could end up adding more *Pods* than you actually need.

### Custom Metrics

....
