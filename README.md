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

## Installing Linkerd 2
Follow the steps provided by the [Official Linkerd 2 documentation](https://linkerd.io/2/getting-started/) to install Linkerd's control plane in your cluster.  
If you want to have automatic proxy injection(recommended) for your functions you also need to follow [this tutorial](https://linkerd.io/2/tasks/automating-injection/) to enable it. Note that if you don't enable this then you will have to manually add the proxy to each new function you deploy.

## Adding Linkerd 2 to OpenFaaS
Injecting Linkerd 2 to an existing deployment is very simple. It only requires one command. First lets inject it to the gateway:
```
$ kubectl -n openfaas get deploy gateway -o yaml | linkerd inject --skip-outbound-ports=4222 - | kubectl apply -f -
```
This command injects the linkerd side-car and enables the port `4222` for outbound traffic. This port is used to communicate with `NATS`.

Since the functions will be added dynamically we'll enable the automatic injection on the `openfaas-fn` namespace. To do this we simply need to annotate the namespace:
```
$ kubectl annotate namespace openfaas-fn linkerd.io/inject=enabled
```
If you already had functions deployed in your namespace you can run the injection on all of them with the following command:
```
$ kubectl -n openfaas-fn get deploy -o yaml | linkerd inject - | kubectl apply -f -
```

#### Linkerd 2 and the Ingress Controller
If you are using an ingress controller you need to do an extra step(if you are using a service type LoadBalancer for your gateway you can skip this).
First we need to inject the linkerd proxy into the nginx ingress controller:
```
$ kubectl -n default get deploy <name of your ingress controller> -o yaml | linkerd inject - | kubectl apply -f -
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


## Dashboards
Linkerd 2 comes with a very helpful and detailed dashboards that allow us to look into the traffic going through our cluster. To open the dashboard run:
```
$ linkerd dashboard &
```
This will open up the browser, from there you can click the `Tap` option in the side bar and look for traffic flowing from the gateway to the functions. For example:

Traffic going from the ingress controller to the gateway and finally from the gateway to the function:
![gateway-to-function-traffic](/docs/gateway-dashboard-with-ingress.png)
Incoming traffic to a function:
![incoming-traffic-to-a-function](/docs/list-function-linkerd-request.png)