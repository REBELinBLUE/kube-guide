# Extended Tutorial

Now that you have followed through a step by step tutorial, we will discuss some more advanced features. In this 
tutorial we will cover the following topics

- Editing and Patching Resources
- Using *HorizontalPodAutoscalers*
- Applying *ResourceQuota* and *LimitRange*
- Using *NetworkPolicy*
- Using *PodSecurityPolicy* and *PodDisruptionBudget*
- Using multiple clusters
- Patching
- *ServiceAccounts*, *Roles* and *RoleBindings*

## Editing and Patching Resources

Until now we have updated resources by changing our manifest files and using `kubectl apply` to apply the new version, but there are a couple of other ways to update resources.

The first way is using the `edit` command, you can use this to edit a specific instance of a resource or all resources of a type; when you run the command the file will be opened in the editor defined by your shell's `$KUBE_EDITOR` or `$EDITOR` variables (normally `vi` if not set).

Let's try to edit the kuard *Deployment*.

```bash
❯ kubectl edit deployment/kuard
```

When the file opens you will notice there are a lot more fields than we originally created the *Deployment* with. The`status` field is managed by *Kubelet* and can't be edited, other fields are default values. Kubernetes has a concept of [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) which run when you submit a resource to the API, prior to the resource being created. There are several built in controllers but you are also able to define your own. There are 2 types of *Admission Controllers*, the first *MutatingAdmissionWebhook* makes changes to the resource such as sets defaults, the second *ValidatingAdmissionWebhook* checks the resource and either accepts or rejects it.

For example, in the previous tutorial we mentioned how *Pods* have default *tolerations* for *Node* health, these areset by the `DefaultTolerationSeconds` *Admission Controller*.

Change the image tag in the editor from `purple` to `green` and then exit the editor, the resource will be changed.

```bash
❯ kubectl edit deployment/kuard
deployment.apps/kuard edited
```

if you make a mistake Kubernetes will reject the changes

```bash
❯ kubectl edit deployment/kuard
error: deployments.apps "kuard" is invalid
```

This is all well and good for a real person, although it is less than ideal (really you should keep your manifests under version control and apply the changed version), but what if you want to change a resource programmatically, for example as part of a CI system?

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

For the following examples you need the [jq](https://stedolan.github.io/jq/) command installed, it is available via [brew](https://brew.sh) if you are using a Mac.