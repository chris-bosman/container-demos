# Kubernetes Demo #

Okay, we've created our own Docker images, and we've gone over some of what Kubernetes brings to the table. But let's see it in action. What can Kubernetes do with what we have and make it better?

## Setup ##

First, let's set up a Kubernetes cluster on our local machine using minikube. You can find the documentation on that installation [here](https://kubernetes.io/docs/tasks/tools/install-minikube/). Minikube is great for testing our Kubernetes deployments locally before pushing them to Azure.

We will also need to install kubectl, the command line tool for Kubernetes. The minikube installation page linked above includes a link for that step as well.

Once we have both those up and running, we can start our minikube Kubernetes instance:

```console
$ minkube start
minikube config set WantUpdateNotification false
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

One **extremely** important note regarding using kubectl is **context awareness**. kubectl only manages a single Kubernetes cluster at a time, and uses what it calls context to determine which cluster you're intending to run commands on. If you're not entirely sure what context you're running in, your best friend is the command `kubectl config current-context`. This command will give you the name of the Kubernetes cluster kubectl is currently paired to.

```console
$ kubectl config current-context
minikube
```

Good, we're in our minikube context. We can move on.

## Deployments ##

Kubernetes configuration is done best via declarative YAML files. Technically, you can use kubectl and dictate all your configuration along the command line, but that is very error-prone and inefficient, so pretend I didn't say that. In this folder, I have some demo YAML files based off of our node-hello-world image we created in the Docker demo. Let's start using them.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-hello-world
  labels:
    app: node-hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-hello-world
  template:
    metadata:
      labels:
        app: node-hello-world
    spec:
      imagePullSecrets:
      - name: acr-auth
      containers:
      - name: node-hello-world
        image: predictiveindexacr.azurecr.io/node-hello-world:latest
        imagePullPolicy: Always
        ports:
          - containerPort: 3000
            name: app-port
        livenessProbe:
          exec:
            command:
            - curl
            - localhost:3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

Let's to section-by-section and look at this Deployment file to understand what's happening.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-hello-world
  labels:
    app: node-hello-world
```

In Kubernetes, `apiVersion`, `kind`, and `metadata` are required fields for every resource. The API version tells the Deployment what APIs to talk to on the Control Plane. That API version must match the available API versions for the resource "kind". Metadata is self-explanatory, but the concept of "labels" are of increasing importance in the Kubernetes world and worth going into more detail about.

Pods in Kubernetes are temporary. They're intended to be temporary and they're handled as if they are going to be temporary. They can be killed off for any number of reasons and reborn for myriad more. As such, IP addresses and names within the cluster are constantly changing, and are handled dynamically. Thus, referring within a cluster to resources by IP address or DNS name is a bad practice, as there's no guarantee that said IP address or DNS name will even exist by the time the next request comes around.

Kubernetes' solution to this is labels. By giving resources labels in the YAML file, that label is static and every instance of that resource will have that label. Thus, it can be referred to in perpetuity by the cluster without risking absence. It's thus best practice to label every resource. At this point, we have only labeled the Deployment itself (which is essentially a parent resource that tells Kubernetes to spawn Replication Controllers and Pods).

Alright, let's move on.

```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-hello-world
```

The `replicas` field here is dictating how many Pods should be spun up upon deployment, while the `selector` field is ensuring the Deployment is going to register only Pods with the node-hello-world `label`.

```yaml
  template:
    metadata:
      labels:
        app: node-hello-world
```

Below the `template` property is where we define the actual configuration for our Pod. It has its own `metadata` field where we tell the Pods themselves to receive the correct `label`. Again, because labels are so important, these all have to be correct, otherwise the Deployment won't know how many pods it has spun up, among other issues.

```yaml
    spec:
      imagePullSecrets:
      - name: docker-auth
      containers:
      - name: node-hello-world
        image: index.docker.io/[docker-username]/node-hello-world:latest
        imagePullPolicy: Always
        ports:
          - containerPort: 3000
            name: app-port
        livenessProbe:
          exec:
            command:
            - curl
            - localhost:3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

I'm going to skip over `imagePullSecrets` and come back to it in a moment. Under the `containers` property, we see our container configuration. We see we've given it the appropriate `name`, that we're using the `image` we created in our Docker demo, and that we're publishing it on the appropriate `containerPort`. 

There's also some additional configuration. `imagePullPolicy` tells our Kubernetes cluster when and if to pull a new copy of an image. Generally and by default, Kubernetes tries *not* to pull container images if it can avoid it. Thus, when using a generic pod sub-tag like "latest", if Kubernetes already has an image that uses that sub-tag, it will not pull in a new image unless explicitly told to do so. This is useful for applications where configuration, state, etc. are handled outside of the pod (say, in a filesystem resource like a Persistent Volume) and the code within the image is largely static. For more dynamically changing software, where configuration likely lives in the pod itself, we want to ensure we're always getting the latest code, and so we set our `imagePullPolicy` to `Always`.

The `livenessProbe` is a Pod specification item that Kubernetes uses to tell if the Pod is operating properly. While there are various infrastructure-level health checks that Kubernetes polls automatically to determine health, these are obviously not specific to the deployed software. `Probes` exist in Kubernetes to allow us to determine if our code is running properly. This example uses an `exec` `command` `Probe`, which runs a bash (`curl localhost:3000`) command on the Pod and, if it returns a non-zero error code, kills the Container and tries again. It has an `initialDelay` of 5 seconds, and then runs again every additional 5 seconds, as defined by the `periodSeconds`.

Okay, let's get back to `imagePullSecrets`. If Kubernetes is pulling from a *publc* Container Registry, this field is not necessary. However, by using a *private* Container Registry, we need to ensure Kubernetes can authenticate to it. Without an `imagePullSecrets` field, our attempet to pull the container image will fail and our Pods will not start. So, before we can attempt to apply this YAML file, we will need to create a `Secret`.

## Secrets ##

Secrets in Kubernetes are resources that are handled in a very specific way. First of all, they're stored in a specific namespace within the etcd cluster that is uniquely encrypted. This is to prevent unauthorized access to your Secrets. Second of all, data read from secrets is base64 *decoded* by Kubernetes on application, meaning the raw data inside a Secrets YAML file must be base64 *encoded*, as without this Kubernetes will not receive the correct values.

Okay, let's look at our Docker-Secret.yaml file. If you open it up, there's a lot in there, but we just want to focus on is below

```yaml
...
- apiVersion: v1
  data:
    .dockerconfigjson: [Redacted]
  kind: Secret
  metadata:
    ...
    name: docker-auth
    ...
  type: kubernetes.io/dockerconfigjson
...
```

As we did in the Deployment file, we see fields for `apiVersion`, `kind`, and `metadata`. However, in this case, we don't have any labels assigned under `metadata`. Why is this? Well, it's because this particular secret is not intrinsically tied to the node-hello-world app. It may, in fact, be used for other things within the cluster. If we want to run, say, a delete command for the node-hello-world app resources, but leave resources that are tied to other app things within the environment alone, we don't want to assign this particular resource the node-hello-world label.

Kubernetes expects to receive Docker Credential Secrets in a very specific way, as a base64-encoded string representation of the file ~/.docker/config.json that is created when you log into a Container Registry. This is what is referred to in our Secrets file as `type: kubernetes.io/dockerconfigjson`. However, the authentication information that was once included in that config.json is now obfuscated. To get around this, Kubernetes allows us to generate this entire, properly formatted YAML file, via a kubectl command.

```console
$ kubectl create secret docker-registry docker-auth --docker-server=https://index.docker.io/v1/ --docker-username=[docker-username] --docker-password=[docker-password] --docker-email=[docker-email]
```

We should now be able to see the secret in our cluster.

```console
$ kubectl get secrets docker-auth
NAME          TYPE                             DATA      AGE
docker-auth   kubernetes.io/dockerconfigjson   1         8m
```

kubectl uses either of the options "create" or "apply" to turn a YAML file into a resource within the Cluster. However, apply is recommended, as it preps the Kubernetes cluster that the Resource may be changed at a later date (say, when credentials are changed) via another apply command.

Another nice thing about Kubernetes is that you can export any existing resource to a YAML file, so we can take the secret we just generated and use it in other Deployments like this:

`$ kubectl get secret docker-auth -o yaml >> Docker-Secret.yaml`

## Deployments, pt. 2 ##

Now that our prerequisite Secret is in the cluster, let's deploy our Pods.

```console
$ kubectl apply -f ./Deployment.yaml
deployment.apps/node-hello-world created
```

And let's watch those pods get started. The `-w` flag below is kubectl's "watch" flag, and it watches the out put of a kubectl command until it changes. To exit watching, press CTL-C.

```console
$ kubectl get pods -w
NAME                                READY     STATUS              RESTARTS   AGE
node-hello-world-5f4fbc57b5-7lfh6   0/1       ContainerCreating   0          9s
node-hello-world-5f4fbc57b5-7lfh6   1/1       Running   0         25s
^C
```

Awesome! Our pod is up and running. Let's hit http://localhost:3000/ and check it out!

![ErrorPic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/master/error-pic.jpg)

Ugh. Not again. So, what's happening now?

## Services ##

Well, just as it requires special handling from Docker to expose a port outside of the container, the same principal holds in Kubernetes. In the case of Kubernetes, we need to create an Endpoint. An Endpoint is the pairing of a Pod to a Service.

Let's look at the Service.yaml file in our repo to get an idea of what it does.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-hello-world
spec:
  ports:
  - port: 3030
    protocol: TCP
    targetPort: 3000
  selector:
    app: node-hello-world
  type: ClusterIP
```

Again, like all Kubernetes resources, we see `apiVersion`, `kind`, and `metadata`. Under `spec` we're using the `app` label as our `selector` to ensure that Kubernetes knows to apply this Service to all Pods with the `node-hello-world` label. But let's take a closer look at the `ports` `spec`.

```yaml
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 3000
```

As we know, we're running our node-hello-world application on port 3000. We can, of course, simply dictate the Service run on the same port, but to demonstrate the capabilities of services, we're actually using the Service resource to proxy our application to port 80, by defining port 80 as our service's `port` while targeting the Pod's `targetPort` of 3000.

Regarding `type`, there are a handful of different types of Services, with `ClusterIP` being the most common. You can read more about the other types of Services [here](https://kubernetes.io/docs/concepts/services-networking/#publishing-services-service-types).

Let's apply our Service to our cluster and see what happens.

```console
$ kubectl apply -f ./Service.yaml
service/node-hello-world created
$ kubectl get svc
NAME               TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
kubernetes         ClusterIP   10.96.0.1     <none>        443/TCP    104d
node-hello-world   ClusterIP   10.96.203.3   <none>        3000/TCP   4s
```

Can we hit our http://localhost site yet?

![ErrorPic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/master/error-pic.jpg)

Sad face. What's happening now?

## Port-Forward ##

Well, what did our Service do? It proxyed port 80 to port 3000, but *only within our Kubernetes cluster*. We still have not actually exposed that Service externally to our cluster. So, what's the point of a Service? To create a single entrypoint for however many Pods may be running in our cluster. Without a Service, we would have to be connecting to our Pods directly, which defeats the whole purpose of clustering. The Service serves as a single ingress to multiple Pods.

So, how do we verify that our application is working inside the cluster before we try to expose it? Well, kubectl has a `port-forward`function that serves the same purpose as docker's `-p` flag. Let's give that a try.

```console
$ kubectl port-forward deployment/node-hello-world 3000:3000
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
Handling connection for 3000
Handling connection for 3000
Handling connection for 3000
```

A couple things to note here in the command `kubectl port-forward deployment/node-hello-world 3000:3000`. The first is that we are referencing the overarching Deployment resource in our command, even though the port-forward command accepts naming pods directly. Why is this? Well, remember our pod name?

`node-hello-world-5f4fbc57b5-7lfh6`

That randomly-generated string at the end is a pain to type and remember. Meanwhile, the name for a Deployment remains static according to our YAML definition and is thus much easier to reference in the CLI.

The second thing to note is that we're bypassing our Service resource here to go directly to some pod, chosen by the Deployment resource. This means we're not port-forwarding port 80, as defined in our Service, but rather port 3000, as defined by our Deployment. So, let's go to http://localhost:3000 and see how we're doing.

![hello-world-image](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/master/hello-world.jpg)

Cool!

## Ingress ##

But, of course, we still want to expose our service *outside* of our cluster. So, how do we do that? That's where the Kubernetes Ingress resource comes in. Ingress is the Resource that allows external users to talk to our Service and, in pair, our Pod.

### Ingress Controller ###

Ingress, though, is not automatically handled by a newly-deployed Kubernetes cluster. It requires a new type of Controller running in our Control Plane. In this case, an Ingress Controller. There are many types of Ingress controllers, and you can even write your own. In this demo, we're going to be using the relatively simple nginx ingress controller, which minikube supports out-of-the-box.

```console
$ minikube addons enable ingress
ingress was successfully enabled
```

By default, the nginx ingress is typically installed into a separate namespace within a Kubernetes cluster, usually the kube-system namespace, where the rest of the control plane resides. To view things running in this namespace, we need to explicitly call it in our kubectl commands using the `-n` flag.

```console
$ kubectl get pods -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
default-http-backend-kkn9g              1/1       Running   0          2m
etcd-minikube                           1/1       Running   0          3h
kube-addon-manager-minikube             1/1       Running   1          104d
kube-apiserver-minikube                 1/1       Running   0          3h
kube-controller-manager-minikube        1/1       Running   0          3h
kube-dns-86f4d74b45-cslws               3/3       Running   4          104d
kube-proxy-7jngz                        1/1       Running   0          3h
kube-scheduler-minikube                 1/1       Running   0          3h
kubernetes-dashboard-5498ccf677-jxdv7   1/1       Running   3          104d
nginx-ingress-controller-lmctk          1/1       Running   0          2m
storage-provisioner                     1/1       Running   3          104d
```

The items to take note of here are the nginix-ingress-controller Pod, which will handle all incoming requests from Ingress resources in the Pod (so long as they reference the nginx Ingress), and the default-http-backend Pod. In our Ingress, we will route traffic according to `paths`. Any traffic that does not fit an explicitly defined path will reroute to the default-http-backend Pod, which by default simply displays a 404 message.

### Ingress Resource ###

Okay, now that we have an Ingress Controller, let's look at our Ingress resource.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: node-hello-world
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
    kubernetes.io/ingress.allow-http: "false"
spec:
  rules:
  - host: demo.kubedemo.com
    http:
      paths:
      - backend:
          serviceName: node-hello-world
          servicePort: 80
        path: /
  tls:
  - secretName: ssl-cert
    hosts:
    - demo.kubedemo.com
```

Yep, `apiVersion`, `kind`, `metadata`, all there as expected. Ingress resources don't accept `label` metadata, so we are not doing that here. But what are `annotations`? The Kubernetes documentation refers to them as key/value maps that, unlike labels, "are not used to identify and select objects." And also notes that "The metadata in an annotation can be small or large, structured or unstructured, and can include characters not permitted by labels."

This doesn't seem to mean much, aside from simply applying metadata. However, in the case of Ingress, annotations are hugely important. They are used by Ingresses to dictate actual configuration to the Ingress Controller. It is actually annotations that will dictate how traffic is handled to our cluster from the outside.

### Annotations ###

Let's take a closer look at these annotations and go over what they mean.

```yaml
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
    kubernetes.io/ingress.allow-http: "false"
```

* `kubernetes.io/ingress.class: nginx`: This annotations tells incoming traffic that we're going to be using nginx as our Ingress Controller
* `nginx.ingress.kubernetes.io/rewrite-target: /`: This annotation dictates how to rewrite the external-facing URL from the internal page.
* `nginx.ingress.kubernetes.io/auth-type: basic`: To demonstrate how we can use our Ingress for security, we're using this annotation to tell our Ingress that basic authentication (i.e. username/password) is going to be required.
* `nginx.ingress.kubernetes.io/auth-secret: basic-auth`: This annotation instructs the Ingress Controller on which Secret resource to check the input credentials against.
* `nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'`: This annotation tells the Ingress what to display to the user when they attempt to access the site and receive a username/password prompt.
* `kubernetes.io/ingress.allow-http: "false"`: Finally, this annotation tells the Ingress to automatically reroute all HTTP requests to HTTPS.

Note that the nginx Ingress has several different auth-types available, including OAuth2 integration and certificate-based authentication.

So, if we want to use basic authentication, and we're telling Kubernetes to use a Secret resource to do so, how do we handle that? We can use the `htpassword` command to create a file that kubectl then understands how to convert into a Secret.

```console
$ htpasswd -c auth admin
New password: [password]
Re-type new password:
Adding password for user admin
$ kubectl create secret generic basic-auth --from-file=auth
secret "basic-auth" created
$ kubectl get secret basic-auth -o yaml >> Ingress-Secret.yaml
```

### Ingress Spec ###

Okay, now let's look at our `spec`:

```yaml
spec:
  rules:
  - host: demo.kubedemo.com
    http:
      paths:
      - backend:
          serviceName: node-hello-world
          servicePort: 80
        path: /
```

As you can see, we use our `host` field to dictate the URL we'll be accepting requests on. This, in conjunction with our `path` field here, tell the Ingress Controller whether to send traffic to our app's Pod or the http-default-backend pod. In this case, if inbound traffic doesn't explicitly attempt to reach demo.kubedemo.com/ or an existing subsite therein, traffic will be sent to our Default Backend instead. Since we don't have any other pages in our node-hello-world app, if people attempt to reach a site like demo.kubedemo.com/dummy.html, they'll be rerouted to that Default Backend. Also note that our `servicePort` is tied to the `port` of our Service, not our Pod.

### TLS ###

Okay, let's look at the final portion of our Ingress.yaml file.

```yaml
  tls:
  - secretName: ssl-cert
    hosts:
    - demo.kubedemo.com
```

Okay, we're referencing another secret. But what for? Well, to secure our website with `tls`. We don't want Google telling everybody we're insecure, do we? But, again, we now have a Secret we need to create before our resource will work. So how do we create a Secret for TLS? Thankfully, Kubernetes can convert certificates into Secrets, and then the Ingress is able to understand them as certs.

Obviously, in production environments we'll use certificates generated by an actual Certificate Authority, but for the purposes of our demo we can use a self-signed certificate.

```console
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=demo.kubedemo.com"
Generating a 2048 bit RSA private key
..............................+++
................+++
writing new private key to '/tmp/tls.key'
-----
$ kubectl create secret tls ssl-cert --key /tmp/tls.key --cert /tmp/tls.crt
secret/ssl-cert created
$ kubectl get secret ssl-cert
NAME       TYPE                DATA      AGE
ssl-cert   kubernetes.io/tls   2         23s
$ kubectl get secret ssl-cert -o yaml >> SSL-Secret.yaml
```

So long as you have the (unencrypted) .key file and the .crt file for your certificate, the `kubectl create secret` command above can be used to import that SSL Certificate into your cluster for use by your Ingress.

Now, let's add our Ingress Resource.

```console
$ kubectl apply -f ./Ingress.yaml
ingress.extensions/node-hello-world created
```

After a few minutes, our Ingress Controller will pick up the changes to our Ingress. To see this, we can use the `kubectl logs` command:

```console
$ kubectl logs -l "app=nginx-ingress-controller" -n kube-system
...
I0802 19:21:53.243618       5 controller.go:183] backend reload required
I0802 19:21:53.366594       5 controller.go:192] ingress backend successfully reloaded...
```

The `logs` command exposes the application logs of any running pod and, as shown above, you can use the label selector flag `-l` to filter your logs and avoid having to remember the randomly-generated pod names. Once you see the above output from your logs command, you know the Ingress Controller has picked up your changes and applied TLS successfully.

### Access ###

As mentioned, our Ingress Controller is specifically looking for the URL input `demo.kubedemo.com`. Because we can't publish our minikube instance on the Internet, we'll hack around this by writing our hosts file with the IP address of the minikube instance.

```console
$ echo "$(minikube ip) demo.kubedemo.com" | sudo tee -a /etc/hosts
Password:
[minikube ip] demo.kubedemo.com
```

And now, let's see what happens when we hit https://demo.kubedemo.com.

![SSL-Pic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/ssl-pic.jpg)

This SSL warning lets us know that, though the Cert Authority is considered invalid (because we self-signed it), that our Ingress *is* using our Certificate on our site. That's a win! Let's click through this and see what happens.

![Auth-Pic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/auth-pic.jpg)

Awesome, we're getting prompted for the Username and Password we described in our Ingress auth-type! Let's go ahead and enter those credentials.

![Kube-Pic](https://raw.githubusercontent.com/chris-bosman/Public-Scripts/kube-hello-world.jpg)

Hooray! We did it!

## Horizontal Pod Autoscalers ##

But, of course, that's not all. There's other Kubernetes resources we can apply to get the functionality we want in our app that were referenced earlier. We can go into more details on those on demand, but I do want to cover Horizontal Pod Autoscalers. This resource is a huge benefit of Kubernetes, as it allows Kubernetes the power to scale up and down the nodes of our app automatically based on defined metrics. Let's take a look at our HPA.yaml file.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: node-hello-world
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-hello-world
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
```

At this point, it's likely you can understand all of the fields presented here. The spec is referring to the Deployment we created, so the Autoscaler knows which Pods to scale. `minReplicas` and `maxReplicas` define the lowest and highest amount of Pods that can exist of the Deployment's type. And, under `metrics`, we're polling `cpu` usage and making sure it doesn't reach over 50%.

CPU and Memory are the only resources that Kubernetes supports natively for HPA polling. However, you can create custom metrics within Kubernetes, as well as polling External metrics from a monitoring system. These scenarios are outside the scope of this demo.

We're not going to deploy this into our minikube, as minikube doesn't support autoscaling, but it's useful to understand how the HPA functions.

## Conclusion ##

This demo covered a *lot* of ground, and it can seem extremely complex. As such, I want to really pare down what was actually done here to demonstrate how easy this is once we understand what we're doing. Here's the entirety of commands that we ran to go from our Docker image to a working Kubernetes cluster, using a Docker image from a private registry, with TLS and password authentication enabled:

    # Create cluster
    minikube start

    # Create auth for private docker image
    kubectl create docker-registry docker-auth —docker-server=https://index.docker.io/v1/ —docker-username=[docker-username] —docker-password=[docker-password] —docker-email=[docker-email]

    # Create Pods
    kubectl apply -f ./Deployment.yaml

    # Create Service
    kubectl apply -f ./Service.yaml

    # Enable Ingress
    minikube addons enable ingress

    # Create username/password for app authentication
    htpasswd -c auth admin
    kubectl create secret generic basic-auth --from-file=auth

    # Create SSL cert
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=demo.kubedemo.com"
    kubectl create secret tls ssl-cert --key /tmp/tls.key --cert /tmp/tls.crt

    # Create Ingress Resource
    kubectl apply -f ./Ingress.yaml

    # Write Hosts file
    echo "$(minikube ip) demo.kubedemo.com" | sudo tee -a /etc/hosts

That's 11 lines of code to go from nothing to a working Kubernetes cluster, some of which aren't necessary in a production environment, and some of which create Resources that can be reused for multiple clusters. But, of course, we can still do better...

## Cleanup ##

You can clean up your cluster by running `minikube delete`, but also remember to edit your hosts file do no longer include your entry for demo.kubedemo.com.