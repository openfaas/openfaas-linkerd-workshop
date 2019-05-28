Guide for OpenFaaS / Linkerd2
===

[Linkerd 2](https://linkerd.io/2/) is a service mesh that provides features such as:
* Automatic TLS between pods,
* Dashboard and pre-configured grafana dashboards with prometheus metrics,
* Works along-side ingress controllers,
* Retries and timeouts,
* Telemetry and monitoring,
* A lot more! [Checkout the documentation](https://linkerd.io/2/features/)

One of the goals for Linkerd 2 is that *it just works*. Is an operator-friendly project just like OpenFaaS! :smile:


Today we will install Linkerd to our Kubernetes cluster so that the communication between the Gateway and the Functions goes through the linkerd proxy. These will give us encrypted communication, retries, timeouts and more.

Be sure to have a working OpenFaaS installation on top of a Kubernetes cluster.

If you are running on a GKE cluster with RBAC enabled you need to grant a cluster role of cluster-admin to your Google account:
```bash
kubectl create clusterrolebinding cluster-admin-binding-$USER \
    --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```

## Installing Linkerd 2

Installing Linkerd is easy. First, you will install the CLI (command-line interface) onto your local machine. Using this CLI, you’ll install the Linkerd control plane into your Kubernetes cluster. Finally, you’ll “mesh” one or more services by adding the data plane proxies. (See the [Architecture page](https://linkerd.io/2/reference/architecture/) for details.)

### Step 1: Install the CLI
Install linkerd's CLI if you don't already have it:
```bash
curl -sL https://run.linkerd.io/install | sh
```
Add linkerd to your path:
```bash
export PATH=$PATH:$HOME/.linkerd2/bin
```
Verify that it all worked correctly:
```bash
linkerd version
```

### Step 2: Validate your Kubernetes cluster
Linkerd's CLI has a handy command called `check`. This command allows us to see if our kubernetes cluster meets linkerd's needs:
```bash
linkerd check --pre
```

### Step 3: Install Linkerd 2 onto the cluster
Now lets install linkerd's control plane in our cluster with automatic injection enabled:
```bash
linkerd install --proxy-auto-inject | kubectl apply -f -
```
This may take a minute or two. We can validate that everything's happening correctly by running:
```bash
linkerd check
```

## Adding Linkerd 2 to OpenFaaS
Injecting Linkerd 2 to an existing deployment is very simple. It only requires one command. First lets inject it to the gateway:
```bash
kubectl -n openfaas get deploy gateway -o yaml | linkerd inject --skip-outbound-ports=4222 - | kubectl apply -f -
```
This command injects the linkerd side-car and enables the port `4222` for outbound traffic. This port is used to communicate with `NATS`.

Since the functions will be added dynamically we'll enable the automatic injection on the `openfaas-fn` namespace. To do this we simply need to annotate the namespace:
```bash
kubectl annotate namespace openfaas-fn linkerd.io/inject=enabled
```
If you already had functions deployed in your namespace you can run the injection on all of them with the following command:
```bash
kubectl -n openfaas-fn get deploy -o yaml | linkerd inject - | kubectl apply -f -
```

#### Linkerd 2 and the Ingress Controller
If you are using an ingress controller you need to do an extra step(if you are using a service type LoadBalancer for your gateway you can skip this).
First we need to inject the linkerd proxy into the nginx ingress controller:
```bash
kubectl -n default get deploy <name of your ingress controller> -o yaml | linkerd inject - | kubectl apply -f -
```
Once your ingress controller is injected you will notice that you cannot access your gateway. This is because of the following:
> Linkerd discovers services based on the :authority or Host header. This allows Linkerd to understand what service a request is destined for without being dependent on DNS or IPs.
> When it comes to ingress, most controllers do not rewrite the incoming header (example.com) to the internal service name (example.default.svc.cluster.local) by default.

This means we need to make some changes to the ingress rule. If you are using nginx add the following annotation to it:
```
nginx.ingress.kubernetes.io/configuration-snippet: |
  proxy_set_header l5d-dst-override gateway.openfaas.svc.cluster.local:8080;
  proxy_hide_header l5d-remote-ip;
  proxy_hide_header l5d-server-id;
```
For the rest of the ingress controllers [check out Linkerd's documentation](https://linkerd.io/2/tasks/using-ingress/)

**Caveat for Nginx**: It is not possible to rewrite the header in this way for the default backend. Because of this, if you inject Linkerd into your Nginx ingress controller’s pod, the default backend will not be usable.

## Dashboards
Linkerd 2 comes with a very helpful and detailed dashboards that allow us to look into the traffic going through our cluster. To open the dashboard run:
```bash
linkerd dashboard &
```
This will open up the browser, from there you can click the `Tap` option in the side bar and look for traffic flowing from the gateway to the functions. For example:

Traffic going from the ingress controller to the gateway and finally from the gateway to the function:
![gateway-to-function-traffic](/docs/gateway-dashboard-with-ingress.png)
Incoming traffic to a function:
![incoming-traffic-to-a-function](/docs/list-function-linkerd-request.png)