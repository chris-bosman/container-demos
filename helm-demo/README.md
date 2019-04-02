# Helm Demo #

So, we've now gone over Docker and Kubernetes. Now we want to see how Helm can further reduce our overhead. First, let's install the Helm CLI tool. To do this, follow [this link](https://docs.helm.sh/using_helm/#installing-helm) and follow the appropriate steps for your OS.

## Getting Started ##

Before we discuss custom Charts, let's discussed the preconfigured Charts that you have access to immediatley upon installing Helm. These Charts come from the official Kubernetes Charts repository called "stable". Let's look at what Charts this repository has.

```console
$ helm search
NAME                                   Chart VERSION    APP VERSION                 DESCRIPTION
stable/acs-engine-autoscaler           2.2.0            2.1.1                       Scales worker nodes within agent pools
stable/aerospike                       0.1.7            v3.14.1.2                   A Helm Chart for Aerospike in Kubernetes
stable/anchore-engine                  0.2.0            0.2.3                       Anchore container analysis and policy evaluation engine s...
stable/apm-server                      0.1.0            6.2.4                       The server receives data from the Elastic APM agents and ...
stable/ark                             1.1.0            0.9.0                       A Helm Chart for ark
stable/artifactory                     7.2.2            6.0.0                       Universal Repository Manager supporting all major packagi...
stable/artifactory-ha                  0.2.2            6.0.0                       Universal Repository Manager supporting all major packagi...
...
```

Wow, there's a lot. Look through the various Charts and you can see Charts for familiar platforms like Drupal, Jenkins, MySQL, Minecraft (!!!), Redis, Selenium, and many others. All of these Charts reference a directory in the official Kubernetes Github Chart repository, which can be found [here](https://github.com/helm/Charts/tree/master/stable).

## Helm Prerequisites ##

First, let's start a new minikube cluster.

```console
$ minikube start
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```

Before we run our first Helm command, we have to be aware of one thing. With versions of Kubernetes >1.9, Role-Based Access Control (RBAC) is enabled by default. If we initialize Helm without being aware of this, the resources will be installed, but unable to manage anything within our cluster. To proactively avoid this, we should create an RBAC account for the Helm service, which is called Tiller.

Let's look at the YAML file for this, as we can use it to discuss some aspects of Kubernetes cluster security.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

As you can see we are actually deploying two resources with one YAML file here. This is possible with Kubernetes, but generally not recommended. The reason this is not best practice is because flexible updating of individual components is one of the benefits of a distributed system like Kubernetes, but if all your resources exist in one YAML file, every update evaluates every component, costing unnecessary time and making your YAML files overly complex. In this, case, however, there is no situation where you would want to deploy one of these resources independently of the other; they only work together.

The `kinds` in play here are `ServiceAccount` and `ClusterRoleBinding`. The `ServiceAccount` kind is fairly self explanatory; it creates an Account in the Kubernetes cluster with zero permissions. The `ClusterRoleBinding` kind takes a little more explaining. As you can see it has a `roleRef` field. There are many `ClusterRoles` within a Kubernetes cluster by default. You can view these by running `kubectl get roles -n kube-system`. You can also create your own roles to more granularly control permissions. Whether you're using an existing role or a custom role, refer to it under the `roleRef` `name` field in the `ClusterRoleBinding`, while also referencing the `ServiceAccount` under `subjects` to tie the permissions for that role to the Account.

In both of these Resources, we're explicitly declaring the `namespace` field as kube-system, as this is the namespace in which Helm installs Tiller. Let's go ahead and install these resources into our cluster.

```console
$ kubectl apply -f tiller.yaml
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

## Initializing Helm ##

Now, we can initialize Helm. Note that we *must* explicitly declare that we want to use the service account that we just created, or our Helm operations will fail.

```console
$ helm init --service-account tiller
$HELM_HOME has been configured at /Users/cbosman/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

In production, we will follow the above security instructions, but for the purposes of our demo, we'll skip that.

So, what did this do? Let's take a look.

```console
$ kubectl get pods -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
...
tiller-deploy-b7f79ccd9-klr8m           1/1       Running   0          16h
```

We now have a pod in our Control Plane called tiller-deploy. This pod is what Helm uses to stage Charts and instruct the rest of the cluster how to deploy them. In Helm parlance, it's simply referred to as Tiller.

## Creating Charts ##

Now that we've initialized helm, what can we do? Well, we can install any Helm Chart we want! But, wait, we're not interested in installing Minecraft Helm Charts (Okay, yes I am, just not right now). We want to create our own Helm Chart and deploy *that*.

If you want to quickly generate the minimum necessary files and folders for a Chart, you can do so by running `helm create [Chart-name]`. This will create the full directory structure referred to in the slides. In this repo, I've provided you with the files necessary. Let's go over them to explore what they do.

### Chart.yaml ###

```yaml
apiVersion: v1
appVersion: "1.0"
description: A Helm Chart for Kubernetes
name: node-hello-world
version: 0.1.0
```

Helm uses the YAML syntax the same as Kubernetes does, and requires many of the same fields. You can see `apiVersion` again here, but in the case of Helm, there is only `v1`. It's important to note that the `name` field here must match the name of your parent folder, otherwise the Chart will not properly package. These are automatically set the same by using `helm create [Chart-name]`, but if you're creating Charts manually, this is good to be aware of. `version` and `appVersion` may seem redundant, but they in fact refer to different things. The `version` field refers to the version *of the Helm Chart itself* while the `appVersion` field refers to the version *of the underlying application*. `description` is an optional field (there are many more optional fields, too) and the `helm create` command defines it as "A Helm Chart for Kubernetes" by default. Everything in this Chart.yaml file is simply metadata for the Chart.

### requirements.yaml ###

Huh? I don't see a requirements.yaml file. Well, that's because in this case, we don't have any any prerequisite requirements for our Chart. However, if our Chart *did* depend on the existence of another Chart or Charts, we could define it/them here with the following syntax:

```yaml
dependencies:
  - name: [Chart-name]
    version: [Chart-version]
    repository: http://example.com/Charts
  - name: [Chart2-name]
    version: [Chart2-version]
    repository: http://another.example.com/Charts
```

You may be wondering how to get this information. The easiest way is to run the following commands:

```console
$ helm search nginx
NAME                        Chart VERSION   APP VERSION DESCRIPTION
stable/nginx-ingress        0.23.0          0.15.0      An nginx Ingress controller that uses ConfigMap to store ...
stable/nginx-ldapauth-proxy 0.1.2           1.13.5      nginx proxy with ldapauth
stable/nginx-lego           0.3.1                       Chart for nginx-ingress-controller and kube-lego
...
$ helm repo list
NAME    URL
stable  https://kubernetes-Charts.storage.googleapis.com
local   http://127.0.0.1:8879/Charts
...
```

The `version` the requirements.yaml file is looking for is the Chart VERSION from the `helm search` command. You can then get the repository URL by matching the repo name from the Chart you're looking at to the URL for that repo from the `helm repo list` command.

Once we write dependencies into this file, we have to run the command `helm dependency update` and it will download the referenced Charts into the ./Charts folder. I haven't found great circumstances or use cases for using requirements as opposed to installing these Charts, separately, however, as I've run into some issues trying to use them. I won't go into those issues at this time, but can address them if necessary.

### ./templates ###

Before I get into the values.yaml file, let's look in the /templates folder. Anything look familiar? Oh, wow, it's all the same file names that we had in our Kubernetes Demo! Let's take a look at our good old Deployment.yaml.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
  labels:
    app: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.replicas.min }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      containers:
      - name: {{ .Values.app.name }}
        image: {{ .Values.registry.url }}/{{ .Values.app.name }}:latest
        imagePullPolicy: Always
        ports:
          - containerPort: {{ .Values.port.internal }}
            name: app-port
      imagePullSecrets:
      - name: docker-auth
```

...Huh. That's different. What's going on here? And if we look at all of our other non-Secrets files, much the same thing has happened ot them. Well, if you're at all familiar with [Go templates](https://godoc.org/text/template), you might be able to infer what's happening above. Basically, what we've done is parameterized our repeated values into Go template variables. So, the question then becames, where are these values being fed from?

### values.yaml ###

*That* is where our values.yaml file comes in. This file is the source that propagates its values down into every file in our ./templates folder. Let's look at it now.

```yaml
app:
  name: node-hello-world
registry:
  url: index.docker.io
  username:
  password:
host:
  url: demo.kubedemo.com
ports:
  external: 80
  internal: 3000
replicas:
  min: 1
  max: 10
metrics:
  type: cpu
  threshold: 50
```

From here, you can see the general format of how our files in the ./template folder are referring to their values: `{{ .Values.app.name }}` substitutes in `node-hello-world`. `{{ .Values.host.url }}` inputs `demo.kubedemo.com`. And so on and so forth. This means that when things need to be changed in our Helm Chart, we only need to make those changes once, and the changes propagate down from the values.yaml file.

### More on Values ###

The Go template format and values files can be used for more than just simple parameter substitituion. Helm Charts support almost all the functions of the [Sprig library](https://godoc.org/github.com/Masterminds/sprig), including string manipulation, format conversion, if/then logic, loop functions, comparison functions, and more. In the context of single-application Helm Charts, these are largely unnecessary, but when dealing with more complex Charts, the flexibility that these functions provide is invaluable.

To get an idea of how these functions can be used in more complex environments, you can go back to Kubernetes' official [Helm Chart directory](https://github.com/helm/Charts/tree/master/stable), choose an application at random, and view *its* values.yaml file. Best practice for Helm Charts has become to store defaults in the base values.yaml file and to explain the non-obvious fields in comments within that yaml. You can then go to that Chart's ./templates folder and see how they're using the Go template language to deal with conditions and more in their Charts.

Another thing to mention regarding values.yaml files, is that the Helm command line by default imports that file into a Helm deployment. But, it also has a `-f` flag to choose a *different* file. This is very useful in use cases such as ours, where we might be deploying to multiple environments, all with different parameters. In this case, instead of trying to use a tool like Octopus to substitute many variables at deployment time, we can create a separate values-[environment].yaml file for each environment, and then tell Octopus to use a different values file per environment. This gives us our source of truth in code, gives us a non-production impacting place to adjust variables and variable substitution, and also allows us to lint, test, and troubleshoot each environment's deployment individually.

We can also override the default values.yaml file right in the command line, and in fact for this installation, we're going to do exactly that. Why?

Well, notice under `registry` how we have blank fields for user name and password. It's not possible to re-use our Docker-Secret from our Kubernetes project like we're re-using our SSL-Secret and Ingress-Secret, so we have to find a way to input our information at runtime without putting it in a Chart. To to this, we have to talk about `helpers`.

### _helpers.tpl ###

The _helpers.tpl file in ./templates allows us to run commands against our values using the Go tenmplate functions. In our case, we are simply defining the data that the Docker-Secret expects to see.

```go
{{- define "imagePullSecret" }}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.registry.url (printf "%s:%s" .Values.registry.username .Values.registry.password | b64enc) | b64enc }}
{{- end }}
```

Now, when we run our Helm command, we can use the `--set` flag to set the values for `registry.username` and `registry.password`.

## Using Helm ##

Okay, so let's say we have correctly created our Helm Chart. How, then do we actually use it? First, we have to make sure Helm understands that our files and folder should actually constitute a Chart. To do this, we go to the root directory of the Chart (in this case ./helm-demo/node-hello-world) and run `helm package -u .` This command tells Helm to package up the files and folders and make them readable by Tiller. Note that the `-u` flag won't do anything in our case-- it's used to tell Helm to read the requirements.yaml file and download the applicable dependencies-- but it's good practice to remember to use it every time you package, so as not to accidentally forget any steps.

```console
$ helm package -u .
No requirements found in .../container-demos/helm-demo/node-hello-world/Charts.
Successfully packaged Chart and saved it to: .../container-demos/helm-demo/node-hello-world/node-hello-world-0.1.0.tgz
```

This, however, is not enough for `helm install` to understand how to deploy this Chart. We also need to `index` our Charts and then publish them to an available repository. To do that, we go to the parent folder of our Helm Chart (./helm-demo/) and run `helm repo index .`

This generates or updates an index.yaml file that will look like this:

```yaml
apiVersion: v1
entries:
  node-hello-world:
  - apiVersion: v1
    appVersion: "1.0"
    created: 2018-08-03T09:19:54.637272298-05:00
    description: A Helm Chart for Kubernetes
    digest: 28903e4e45f6f82eda3d5b71f37daf6ff86a03aaff4c9a4a344d76ce2ad649e6
    name: node-hello-world
    urls:
    - node-hello-world/node-hello-world-0.1.0.tgz
    version: 0.1.0
generated: 2018-08-03T09:19:54.634844846-05:00
```

As you can see, it's indexed the Chart.yaml data from our node-hello-world Chart. If we had multiple Charts in this ./helm-demo/ directory, we'd see entries similar to this for each respective Chart.

Lastly, we have to make this index.yaml file available to a repository. The simplest way to do this is to use the `helm serve` command to turn our local directory into a local Chart server. Because this command will take control of our Terminal, we should open a separate Terminal window to run it.

```console
$ helm serve --repo-path .../container-demos/helm-demo/
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879

```

Back in our primary Terminal window, we can now do a few things. First, to verify that our files in our Helm Chart are all correct, we should move back into our ./helm-demo/node-hello-world/ directory and run `helm lint`.

```console
$ helm linst
==> Linting .
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

Awesome! If we're still worried about if our deployment is going to work, Helm also offers the `--dry-run` flag, on the `install` command, which goes through the whole install process without actually triggering the deployments of our resources. We have to go back to the directory where our index.yaml exists (./helm-demo/) to run this.

```console
$ helm install node-hello-world -n node-hello-world --set registry.username=[docker-username] --set registry.password=[docker.password] --dry-run
NAME:   node-hello-world
```

Note the snippet `-n node-hello-world` here. The `-n` flag gives us the power to manually define a name for the Helm installation. This is useful for two reasons. Firstly, without a dictated installation name, Helm creates its own, meaning it would let you try a failing install over even if some of the resources were created. This is less than ideal. Secondly, if we give it a name, then we can reference that name in future adjustments to the Chart, even in the case of failures, by using `helm update`. `helm update` is a non-destructive way to port in changes to a chart.

So, responding to the `--dry-run` with no errors means it succeeded! Hooray! Now, let's do it for real.

```console
$ helm install node-hello-world -n node-hello-world --set registry.username=[docker-username] --set registry.password=[docker.password]
NAME:   node-hello-world
LAST DEPLOYED: Fri Aug  3 10:06:27 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME         TYPE                            DATA  AGE
docker-auth  kubernetes.io/dockerconfigjson  1     1s
basic-auth   Opaque                          1     1s
ssl-cert     kubernetes.io/tls               2     1s

==> v1/Service
NAME              TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)  AGE
node-hello-world  ClusterIP  10.104.140.185  <none>       80/TCP   0s

==> v1/Deployment
NAME              DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
node-hello-world  1        1        1           0          0s

==> v1beta1/Ingress
NAME              HOSTS              ADDRESS  PORTS  AGE
node-hello-world  demo.kubedemo.com  80, 443  0s

==> v2beta1/HorizontalPodAutoscaler
NAME              REFERENCE                    TARGETS        MINPODS  MAXPODS  REPLICAS  AGE
node-hello-world  Deployment/node-hello-world  <unknown>/50%  1        10       0         0s

==> v1/Pod(related)
NAME                               READY  STATUS             RESTARTS  AGE
node-hello-world-5f4fbc57b5-phm2d  0/1    ContainerCreating  0         0s

```

Now, we simply wait for our Pod to start...

```console
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
node-hello-world-5f4fbc57b5-phm2d   1/1       Running   0          9m
```

Add the minikube IP back to our hosts file...

```console
$ echo "$(minikube ip) demo.kubedemo.com" | sudo tee -a /etc/hosts
Password:
[minikube ip] demo.kubedemo.com
```

Hit https://demo.kubedemo.com...

![SSL-Pic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/ssl-pic.jpg)

Click through the SSL warning...

![Auth-Pic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/auth-pic.jpg)

Input the username/password we used previously...

![Kube-Pic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/kube-hello-world.jpg)

And Huzzah!