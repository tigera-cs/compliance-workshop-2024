
# 2. Policies

## Objective

Now that we've deployed our web store application, in order to comply with the SOC2 CC standard we have to secure it. This means applying Security Policies to restrict access as a much as possible. In this section we will walk through building up a robust security policy for our application.

## Policy Introduction

Kubernetes and Calico Security Policies are made up of three main components that all need to be properly designed and implemented to achieve compliance.

**Labels** - Tag Kubernetes objects for filtering.

**Security Policy** - The primary tool for securing a Kubernetes network. It lets you restrict network traffic in your cluster so only the traffic that you want to flow is allowed. Calico Cloud provides a more robust policy than Kubernetes, but you can use them together – seamlessly.

**Policy Tiers** - A hierarchical construct used to group policies and enforce higher precedence policies that cannot be circumvented by other teams.

## Labels

Instead of IP addresses and IP ranges, network policies in Kubernetes depend on labels and selectors to determine which workloads can talk to each other. Workload identity is the same for Kubernetes and Calico Cloud network policies: as pods dynamically come and go, network policy is enforced based on the labels and selectors that you define.

### Label Examples

First, lets manually apply a label to the multitool pod. 

```bash
kubectl label pod multitool mylabel=true
```

We can then see the resulting label using:

```bash
kubectl get pod multitool --show-labels
```
```bash
NAME        READY   STATUS    RESTARTS   AGE    LABELS
multitool   1/1     Running   0          2d6h   mylabel=true,run=multitool
```

Now lets use this to attach a SOC2 label to our application pods. Rather than apply the label one by one, label all pods in the hipstershop namespace with 'soc2=true' with the following command:

```bash
kubectl label pods --all -n hipstershop soc2=true
```

Then, verify the labels are applied:

```bash
kubectl get pods -n hipstershop --show-labels
```
```bash
tigera@bastion:~$ kubectl get pods -n hipstershop --show-labels
NAME                                     READY   STATUS    RESTARTS   AGE   LABELS
adservice-8587b48c5f-hwhzq               1/1     Running   0          21m   app=adservice,soc2=true
cartservice-5c65c67f5d-mhr4s             1/1     Running   0          21m   app=cartservice,soc2=true
checkoutservice-54c9f7f49f-jzh56         1/1     Running   0          21m   app=checkoutservice,soc2=true
currencyservice-5877b8dbcc-st4km         1/1     Running   0          21m   app=currencyservice,soc2=true
emailservice-5c5448b7bc-h5kl9            1/1     Running   0          21m   app=emailservice,soc2=true
frontend-67f6fdc769-zl9qz                1/1     Running   0          21m   app=frontend,soc2=true
loadgenerator-555c7c5c44-8tr7t           1/1     Running   0          21m   app=loadgenerator,soc2=true
multitool                                1/1     Running   0          16m   run=multitool,soc2=true
paymentservice-7bc7f76c67-9smfk          1/1     Running   0          21m   app=paymentservice,soc2=true
productcatalogservice-67fff7c687-vprc8   1/1     Running   0          21m   app=productcatalogservice,soc2=true
recommendationservice-b49f757f-kmhr4     1/1     Running   0          21m   app=recommendationservice,soc2=true
redis-cart-58648d854-hw8rr               1/1     Running   0          21m   app=redis-cart,soc2=true
shippingservice-76b9bc7465-9nrts         1/1     Running   0          21m   app=shippingservice,soc2=true
```

Now that the pods are labeled, lets start applying policies to them.

## Policy Tiers

Tiers are a hierarchical construct used to group policies and enforce higher precedence policies that cannot be circumvented by other teams. As part of your microsegmentation strategy, tiers let you apply identity-based protection to workloads and hosts. All Calico Enterprise and Kubernetes network policies reside in tiers.

For this workshop, we'll be creating 3 tiers in the cluster and utilizing the default tier as well:

**security** - Global security tier with controls such as SOC2 restrictions.

**platform** - Platform level controls such as DNS policy and tenant level isolation.

**app-hipster** - Application specific tier for microsegmentation inside the application.

To create the tiers apply the following manifest:

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: app-hipstershop
spec:
  order: 400
---
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: platform
spec:
  order: 300
---   
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: security
spec:
  order: 200
EOF
```
> Manifest File: [4.1-policy-tiers.yaml](manifests/4.1-policy-tiers.yaml)

## General Policies

After creating our tiers, we'll apply some general policies to them before we start creating our main policies. These policies include allowing traffic to kube-dns from all pods, passing traffic that doesn't explicitly match in the tier and finally a default deny policy.


```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: platform.platform-default-pass
spec:
  tier: platform
  order: 100
  selector: ""
  namespaceSelector: ""
  serviceAccountSelector: ""
  ingress:
    - action: Pass
      source: {}
      destination: {}
  egress:
    - action: Pass
      source: {}
      destination: {}
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.security-default-pass
spec:
  tier: security
  order: 150
  selector: ""
  namespaceSelector: ""
  serviceAccountSelector: ""
  ingress:
    - action: Pass
      source: {}
      destination: {}
  egress:
    - action: Pass
      source: {}
      destination: {}
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: platform.allow-dns
spec:
  tier: platform
  order: -50
  selector: ""
  namespaceSelector: ""
  serviceAccountSelector: ""
  ingress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - "53"
    - action: Allow
      protocol: UDP
      source: {}
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - "53"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - "53"
    - action: Allow
      protocol: UDP
      source: {}
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - "53"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: StagedGlobalNetworkPolicy
metadata:
  name: default.default-deny
spec:
  tier: default
  order: 1100
  selector: "projectcalico.org/namespace in {'hipstershop','default'}"
  namespaceSelector: ''
  serviceAccountSelector: ''
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```
> Manifest File: [4.2-pass-dns-default-deny-policy.yaml](manifests/4.2-pass-dns-default-deny-policy.yaml)

**Separated Policies**
> Manifest File: [4.x-platform-pass.yaml](manifests/4.x-platform-pass.yaml)

> Manifest File: [4.x-security-pass.yaml](manifests/4.x-security-pass.yaml)

> Manifest File: [4.x-default-deny.yaml](manifests/4.x-default-deny.yaml)

> Manifest File: [4.x-allow-dns.yaml](manifests/4.x-allow-dns.yaml)

### Security Policy

Now that we have our foundation in the Policy Tiers, we need to start applying policy to restrict traffic. The first policy we will apply will only allow traffic to flow between pods with the label of 'soc2=true'. Pods without the 'soc2=true' label will also be able to freely communicate with each other.

We will also add a 'soc2-allowlist' policy because we need a way to allow traffic to the frontend of the application as well as allowing DNS lookups from the SOC2 pods to the kube-dns system. We are also adding the metrics-server ingress port so that the AKS konnectivity pod and other pods can call it for getting node and pod metrics as needed.

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.soc2-allowlist
spec:
  tier: security
  order: 0
  selector: all()
  namespaceSelector: ''
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "frontend"
        ports:
          - '8080'
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: k8s-app == "metrics-server"
        ports:
          - '4443'
  egress:
    - action: Allow
      protocol: UDP
      source: {}
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - '53'
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: k8s-app == "kube-dns"
        ports:
          - '53'
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.soc2-restrict
spec:
  tier: security
  order: 75
  selector: soc2 == "true"
  namespaceSelector: ''
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      source:
        selector: soc2 == "true"
      destination: {}
    - action: Deny
      source:
        selector: soc2 != "true"
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination:
        selector: soc2 == "true"
    - action: Deny
      source: {}
      destination:
        selector: soc2 != "true"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```
> Manifest File: [4.3-soc2-isolation-policy](manifests/4.3-soc2-isolation-policy.yaml)

Now we can verify this is working as expected.

## SOC2 Policy Testing

To test, we'll use our MultiTool pods both inside of the 'hipstershop' namespace and in the default namespace. Before we can complete the testing from the default namespace, we'll have to apply a policy that allows egress traffic from the pods in the default namespace. This is because we're applying an egress policy in an earlier step, so now, if we don't allow it explicitly, the default deny will drop the traffic when it is enforced. To get around this we'll apply this policy:

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: platform.default-egress
  namespace: default
spec:
  tier: platform
  order: 75
  selector: ''
  serviceAccountSelector: ''
  egress:
    - action: Allow
      source: {}
      destination: {}
  types:
    - Egress
EOF
```
> Manifest File: [4.4-default-egress-policy.yaml](manifests/4.4-default-egress-policy.yaml)

For testing we will use two tools, curl and netcat.

> Curl will use the '-I' option to only return the header to keep the output short

> netcat will use the following:
> -z for Zero-I/O mode [used for scanning]
> -v for verbose
> -w 3 to limit the timeout to 3 seconds

Before we start testing, we're going to get the addresses of all the Online Boutique services so we can use them in the testing to follow. To do this we'll run the following command and keep the output handy:

```bash
kubectl get svc -n hipstershop -o wide
```

Example output:
```bash
$ kubectl get svc -n hipstershop
NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
adservice               ClusterIP      10.0.131.26    <none>          9555/TCP       4h37m
cartservice             ClusterIP      10.0.214.232   <none>          7070/TCP       4h37m
checkoutservice         ClusterIP      10.0.162.114   <none>          5050/TCP       4h37m
currencyservice         ClusterIP      10.0.227.232   <none>          7000/TCP       4h37m
emailservice            ClusterIP      10.0.72.153    <none>          5000/TCP       4h37m
frontend                ClusterIP      10.0.41.230    <none>          80/TCP         4h37m
frontend-external       LoadBalancer   10.0.209.155   51.143.16.163   80:30113/TCP   4h37m
paymentservice          ClusterIP      10.0.85.72     <none>          50051/TCP      4h37m
productcatalogservice   ClusterIP      10.0.60.54     <none>          3550/TCP       4h37m
recommendationservice   ClusterIP      10.0.20.46     <none>          8080/TCP       4h37m
redis-cart              ClusterIP      10.0.160.215   <none>          6379/TCP       4h37m
shippingservice         ClusterIP      10.0.77.30     <none>          50051/TCP      4h37m
```

First, from inside of the 'hipstershop' namespace, we'll exec into the multitool pod and connect to the 'frontend' as well as try to connect to the 'cartservice' directly. To do this we will use NetCat and Curl.

From the above output, we know that our 'cartservice' is has an address of '10.0.214.232'.

Exec into the pod:
```bash
kubectl exec -n hipstershop multitool --stdin --tty -- /bin/bash
```

Test connectivity to 'cartservice' directly:
```bash
bash-5.1# nc -zvw 3 10.0.214.232 7070
10.0.214.232 (10.0.214.232:7070) open
```
And connectivity to the 'frontend':
```bash
bash-5.1# curl -I 10.0.41.230
HTTP/1.1 200 OK
Set-Cookie: shop_session-id=1939f999-1237-4cc7-abdb-949423eae483; Max-Age=172800
Date: Wed, 15 Feb 2023 23:54:25 GMT
Content-Type: text/html; charset=utf-8
```

As expected, we can reach both services from a pod with the soc2=true label.

Exit from the shell by typing **exit**.

Now lets try from a pod without the 'soc2=true' label that is outside of the namespace. To do this, we'll use our multitool pod in the default namespace:

```bash
kubectl exec multitool --stdin --tty -- /bin/bash
```

```bash
bash-5.1# nc -zvw 3 10.0.214.232 7070
nc: 10.0.214.232 (10.0.214.232:7070): Operation timed out
```
```bash
bash-5.1# curl -I 10.49.14.192
HTTP/1.1 200 OK
Set-Cookie: shop_session-id=772c5095-11f5-4bb0-9d42-0ef8dcda9707; Max-Age=172800
Date: Wed, 26 Jan 2022 20:21:54 GMT
Content-Type: text/html; charset=utf-8
```

As expected, we can connect to 'frontend' because it has a policy allowing it but we can't connect to the cartservice on 7070 because of our soc2 isolation policy.

Exit from the shell by typing **exit**.

Let's add the 'soc2=true' label to the pod:

```bash
kubectl label pod multitool soc2=true
```

And we can test again:

```bash
kubectl exec multitool --stdin --tty -- /bin/bash
```
```bash
bash-5.1# nc -zvw 3 10.0.214.232 7070
10.0.214.232 (10.0.214.232:7070) open
```

We can successfully connect from the MultiTool pod in the default namespace to a service in the hipstershop namespace as long as they both have the 'soc2=true' label.

## Tenant Isolation

If we need to further isolate the workloads in the cluster, for example in the case of multi-tenancy, we can start isolating based on properties such as namespace or a tenant label. 

Let's restrict access between those 'soc2=true' resources even further starting with isolating the hipstershop namespace from outside traffic other than the frontend traffic.

To accomplish this, we will isolate our tenants within the cluster. In this case, our hipstershop will be a single tenant so we'll attach a tenant label to all of it's pods:

```bash
kubectl label -n hipstershop pod --all tenant=hipstershop
```

Now, we'll create a policy that only allows communication within this tenant label:

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: platform.tenant-hipstershop
spec:
  tier: platform
  order: 50
  selector: tenant == "hipstershop"
  namespaceSelector: ""
  serviceAccountSelector: ""
  ingress:
    - action: Allow
      source:
        selector: tenant == "hipstershop"
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination:
        selector: tenant == "hipstershop"
    - action: Deny
      source: {}
      destination: {}
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```
> Manifest File: [4.5-tenant-isolation-policy.yaml](manifests/4.5-tenant-isolation-policy.yaml)

Before we test our policy, we will need to make some updates to the SOC2 restriction policy. Right now the SOC2 policy allows communication between all the 'soc2=true' pods. We want to pass this decision to the 'platform' tier so we will apply the following update:

**SOC2 Policy Update**
```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.soc2-restrict
spec:
  tier: security
  order: 75
  selector: soc2 == "true"
  namespaceSelector: ''
  serviceAccountSelector: ''
  ingress:
    - action: Pass
      source:
        selector: soc2 == "true"
      destination: {}
    - action: Deny
      source:
        selector: soc2 != "true"
      destination: {}
  egress:
    - action: Pass
      source: {}
      destination:
        selector: soc2 == "true"
    - action: Deny
      source: {}
      destination:
        selector: soc2 != "true"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```
> Manifest File: [4.7-soc2-policy-update.yaml](manifests/4.7-soc2-policy-update.yaml)

There are multiple ways to accomplish this, we could very easily have isolated traffic within the namespace as well. Isolating based on the tenant label in this scenario accomplishes our goal of isolating traffic within the hipstershop application.  We can verify by again testing using curl and netcat:

Testing from outside of the tenant label (Multitool Pod in the default namespace):
```bash
kubectl exec multitool --stdin --tty -- /bin/bash
```
```bash
bash-5.1# nc -zvw 3 10.0.214.232 7070
nc: 10.0.214.232 (10.0.214.232:7070): Operation timed out
```
NetCat fails to connect to the cart service.

```bash
bash-5.1# curl -I 10.49.14.192
HTTP/1.1 200 OK
Set-Cookie: shop_session-id=90b6b2f4-c701-45c4-8a97-bb6ca5302a47; Max-Age=172800
Date: Wed, 26 Jan 2022 20:42:07 GMT
Content-Type: text/html; charset=utf-8
```
But HTTP traffic is still allowed by our 'soc2-allowlist' rule.

Now we've successfully isolated the 'tenant=hipstershop' label but if we exec into pod within the tenant we can still access services that we shouldn't be able to.

## Microsegmentation with Hipstershop

To perform the microsegmentation we will need to know more about how the application communicates between the services. The following diagram provides all the information we need to know:

![application-diagram](images/hipstershop-diagram.png)

After reviewing the diagram we can come up with a table of rules that looks like this:

Source Service | Destination Service | Destination Port
--- | --- | ---
cartservice | redis-cart | 6379
checkoutservice | cartservice | 7070
checkoutservice | emailservice | 8080
checkoutservice | paymentservice | 50051
checkoutservice | productcatalogservice | 3550
checkoutservice | shippingservice | 50051
checkoutservice | currencyservice | 7000
checkoutservice | adservice | 9555
frontend | cartservice | 7070
frontend | productcatalogservice | 3550
frontend | recommendationservice | 8080
frontend | currencyservice | 7000
frontend | checkoutservice | 5050
frontend | shippingservice | 50051
frontend | adservice | 9555
loadgenerator | frontend | 8080
recommendationservice | productcatalogservice | 3550

This results in the following policy which we can now apply to the app-hipstershop tier using:

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.checkoutservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 650
  selector: app == "checkoutservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "frontend"
      destination:
        ports:
          - "5050"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "cartservice"
        ports:
          - "7070"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "emailservice"
        ports:
          - "8080"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "paymentservice"
        ports:
          - "50051"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "productcatalogservice"
        ports:
          - "3550"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "shippingservice"
        ports:
          - "50051"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "currencyservice"
        ports:
          - "7000"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.cartservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 875
  selector: app == "cartservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"||app == "frontend"
      destination:
        ports:
          - "7070"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "redis-cart"
        ports:
          - "6379"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.frontend
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1100
  selector: app == "frontend"
  ingress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        ports:
          - "8080"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "cartservice"
        ports:
          - "7070"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "productcatalogservice"
        ports:
          - "3550"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "recommendationservice"
        ports:
          - "8080"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "currencyservice"
        ports:
          - "7000"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "checkoutservice"
        ports:
          - "5050"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "shippingservice"
        ports:
          - "50051"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "adservice"
        ports:
          - "9555"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.redis-cart
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1300
  selector: app == "redis-cart"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "cartservice"
      destination:
        ports:
          - "6379"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.productcatalogservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1400
  selector: app == "productcatalogservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: >-
          app == "checkoutservice"||app == "frontend"||app ==
          "recommendationservice"
      destination:
        ports:
          - "3550"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.emailservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1500
  selector: app == "emailservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"
      destination:
        ports:
          - "8080"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.paymentservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1600
  selector: app == "paymentservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"
      destination:
        ports:
          - "50051"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.currencyservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1650
  selector: app == "currencyservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"||app == "frontend"
      destination:
        ports:
          - "7000"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.shippingservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1700
  selector: app == "shippingservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"||app == "frontend"
      destination:
        ports:
          - "50051"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.recommendationservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1800
  selector: app == "recommendationservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "frontend"
      destination:
        ports:
          - "8080"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "productcatalogservice"
        ports:
          - "3550"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.adservice
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 1900
  selector: app == "adservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "frontend"
      destination:
        ports:
          - "9555"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app-hipstershop.allow-loadgenerator
  namespace: hipstershop
spec:
  tier: app-hipstershop
  order: 2000
  selector: app == "loadgenerator"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "frontend"
        ports:
          - "8080"
  types:
    - Egress
EOF
```
> Manifest File: [4.6-hipstershop-policy.yaml](manifests/4.6-hipstershop-policy.yaml)

Before this policy will be applied though, we will have to go back and make a modification to our Tenant Isolation Policy to completely enable our microsegmentation. Right now the Tenant Isolation policy allows open communication between pods with the 'tenant=hipstershop' label. We want to pass this decision to the 'app-hipstershop' tier so we will apply the following update:

**Tenant Isolation Policy Update**
```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: platform.tenant-hipstershop
spec:
  tier: platform
  order: 50
  selector: tenant == "hipstershop"
  namespaceSelector: ""
  serviceAccountSelector: ""
  ingress:
    - action: Pass
      source:
        selector: tenant == "hipstershop"
      destination: {}
  egress:
    - action: Pass
      source: {}
      destination:
        selector: tenant == "hipstershop"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```
> Manifest File: [4.8-tenant-isolation-update.yaml](manifests/4.8-tenant-isolation-update.yaml)

Once this is applied, the policy inside of the 'app-hipstershop' tier should apply and give us microsegmentation inside of our application namespace. The Policy Board should show traffic being allowed by most of our policies:

![Policy Board](images/policy-board.png)

## Limiting Egress Access

Now that we've implemented our microsegmentation policy, there's one last type of policy we should apply; a global egress access policy.

A global (DNS) egress access policy allows us to limit what external resources/domains the pods in our cluster can reach. To build this we need two pieces:

1. A GlobalNetworkSet with a list of approved external domains.
2. An egress policy that applies globally and references our GlobalNetworkSet.

First, lets created our list of allowed domains:

```yaml
kubectl apply -f -<<EOF
kind: GlobalNetworkSet
apiVersion: projectcalico.org/v3
metadata:
  name: global-trusted-domains
  labels:
    external-endpoints: global-trusted
spec:
  nets: []
  allowedEgressDomains:
    - facebook.com
    - '*.tigera.io'
    - tigera.io
EOF
```
> Manifest File: [manifests/4.9-global-trusted-domains.yaml](manifests/4.9-global-trusted-domains.yaml)

And now we'll apply our policy into the security tier and have it reference our list of trusted domains we just created.

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.global-trusted-domains
spec:
  tier: security
  order: 112.5
  selector: ""
  namespaceSelector: ""
  serviceAccountSelector: ""
  egress:
    - action: Allow
      source: {}
      destination:
        selector: external-endpoints == "global-trusted"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Egress
EOF
```
> Manifest File: [2.10-egress-dns-policy.yaml](manifests/2.10-egress-dns-policy.yaml)

Now any pod that doesn't have a more permissive egress policy will only be allowed to access 'google.ca' and 'tigera.io' and we can test this with our 'multitool' pod in the 'hisptershop' namespace.

First we'll exec into our multitool pod in the 'hipstershop' namespace:
```bash
kubectl exec -n hipstershop multitool --stdin --tty -- /bin/bash
```

And then we'll try to connect to a few domains (docs.tigera.io, facebook.com, github.com)

 >**Note**: Usually ICMP is a decent enough test, but the AKS LB blocks outbound ICMP to the internet so we'll be using curl instead. [Link to Question](https://learn.microsoft.com/en-us/answers/questions/109959/pinging-external-hosts-from-container-in-aks)
 
 

```bash
bash-5.1# curl -m3 -skI https://docs.tigera.io --show-error --fail | grep HTTP
HTTP/2 200
```

```bash
bash-5.1# curl -m3 -skI https://facebook.com --show-error --fail | grep HTTP
HTTP/2 301
```

>**Note**: The 301 Permanent Redirect response here from Facebook is expected as it redirects to www.facebook.com, the important bit is that we ARE getting a response and that Calico policy isn't denying the outbound flow.

```bash
bash-5.1# curl -m3 -skI https://github.com --show-error --fail | grep HTTP
curl: (28) Connection timed out after 3001 milliseconds
```

Our curl to docs.tigera.io and facebook.com are successful but our curl to github.com is denied and times out.

Exit from the shell by typing **exit**.

## Enforcing the Default Deny Policy

The last step in the policy creation process is to enforce our default-deny policy. Currently, with the default-deny policy in staged mode, any traffic that is passed but does not match another tier will be allowed.

Through the UI, enforce the the default-deny policy by opening it from the Policy Board and select the Edit option. 

<p align="center">
  <img src="images/policy-board-default-deny.png" alt="Default Deny Policy Board" align="center" width="400">
</p>

From the Policy Edit screen, select Enforce.

<p align="center">
  <img src="images/enforce-default-deny.png" alt="Enforce Default Deny" align="center" width="600">
</p>

Now our policies are complete. 

### Reference Documentation

[Calico Cloud - Policy Tiers](https://docs.tigera.io/calico-cloud/network-policy/policy-tiers/tiered-policy)

[Securing Hipstershop Blog](https://www.tigera.io/blog/securing-the-hipster-shop-with-calico-network-policies/)

[:arrow_right:5. Reporting and Visibility](../5.%20Reports/readme.md)<br>

[:arrow_left:3. Deploy Demo Microservices App](../3.%20Deploy%20App/readme.md)

[:leftwards_arrow_with_hook: Back to Main](../README.md)  