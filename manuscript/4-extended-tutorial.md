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

This is known as a "Strategic Merge" because Kubernetes will attempt to decide what to do based on the fields in the patch file. If we had multiple containers defined in this patch the new ones will be added, but if we have multiple defined in the *Deployment* the ones missing from the patch will not be removed.

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

