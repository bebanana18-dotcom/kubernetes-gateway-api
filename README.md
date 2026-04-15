# Kubernetes Gateway API — Complete Notes

---

## Why Does Gateway API Exist?

### The Problem with Ingress

Kubernetes Ingress was the original way to expose HTTP services to the outside world. Simple enough — until the real world showed up and started asking for things like:

- **Rate Limiting** — No native support. Good luck.
- **Web Application Firewall (WAF)** — Also no.
- **Canary Deployments** — Nope.
- ...and many more features that production engineers *actually need*.

### The "Solution" That Made Things Worse

Ingress vendors like **NGINX** and **GCP-GCE** tried to plug the gaps by introducing **annotations**. The spec wasn't enough, so they bolted on custom metadata fields.

But here's the dark joke: **annotations were vendor-specific.**

Doing a canary deployment with GCP-GCE Ingress? Use their annotation.
Switched to NGINX Ingress? Completely different annotation. Start over.

This meant your configs were **non-portable** — tightly coupled to whoever sold you the controller. A temporary "solution" that looked like this:

```json
{
  "Type": "forward",
  "ForwardConfig": {
    "TargetGroups": [
      {"ServiceName": "service-1", "ServicePort": "80", "Weight": 20},
      {"ServiceName": "service-2", "ServicePort": "80", "Weight": 20},
      {"TargetGroupArn": "arn-of-your-non-k8s-target-group", "Weight": 60}
    ],
    "TargetGroupStickinessConfig": {"Enabled": true, "DurationSeconds": 200}
  }
}
```

Yep. That's the mess everyone lived with.

---

### Problem 2: The Two-Engineer Problem (Access Separation)

**Situation:** Large companies have two distinct roles working on the same cluster:

| Role | Responsibility |
|---|---|
| **Platform Engineer** | Installs and maintains infrastructure (Ingress controller) |
| **DevOps / App Engineer** | Configures routing for their applications |

**Task:** Both roles had to edit the **same Ingress file** — which is about as sane as two surgeons sharing one scalpel.

**Action:** The DevOps engineer would update the Ingress in ways that violated the Platform Engineer's expectations. There was no way to enforce who owns what.

**Result:** Chaos, finger-pointing, config drift, and the occasional production outage nobody officially caused.

**What they actually needed:** A clean separation where:
- Platform Engineers own: installation and infrastructure config
- DevOps Engineers own: routing rules and backend config

The only way to achieve this was to **split Ingress into separate resources** with separate ownership.

**Enter: Gateway API.**

---

## Gateway API Overview

Gateway API solves both problems by splitting responsibilities across **3 resources**, all of which are **Custom Resources (CRs from CRDs)** — which is a bonus because they're independently versionable and extensible.

```
┌─────────────────────────────────────────────────────────┐
│              GATEWAY API ARCHITECTURE                   │
│                                                         │
│  Platform Engineer owns:                                │
│  ┌──────────────────┐   ┌──────────────────┐           │
│  │  GatewayClass    │──▶│    Gateway        │           │
│  │  (Controller)    │   │  (Listener/Port)  │           │
│  └──────────────────┘   └────────┬─────────┘           │
│                                   │                     │
│  DevOps Engineer owns:            ▼                     │
│                          ┌──────────────────┐           │
│                          │   HTTPRoute       │           │
│                          │ (Routing Rules)   │           │
│                          └──────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

---

## The 3 Resources

### 1. GatewayClass

**Who manages it:** Platform Engineer

The GatewayClass is essentially "which controller implementation are we using?" — the equivalent of specifying `ingressClassName` in the old world, but as a proper first-class resource.

Supported controllers include:
- NGINX Gateway Controller
- Envoy Gateway Controller
- Traefik Gateway Controller
- ...and others

When the Platform Engineer installs the Gateway Controller, a **GatewayClass resource** is created.

---

### 2. Gateway

**Who manages it:** Platform Engineer / DevOps Engineer (shared boundary)

The Gateway resource declares **which GatewayClass to use** and defines **listeners** (protocol + port). It's the actual entry point into your cluster.

---

### 3. HTTPRoute

**Who manages it:** DevOps / Application Engineer

This is where the real power lives. HTTPRoute handles:

- ✅ Web Application Firewall (WAF) support
- ✅ HTTP Redirect support
- ✅ Traffic Splitting
- ✅ Rate Limiting
- ✅ Canary Deployment

If Ingress was a butter knife, HTTPRoute is a Swiss Army knife that actually has the blade you need.

---

## Step-by-Step Demo

### Prerequisites: Install 2 Things

**Option 1: Envoy Gateway Controller** *(recommended — more stable)*
**Option 2: NGINX Gateway Controller**

We'll use Envoy.

---

### Step 1: Install Envoy Gateway via Helm

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.1 \
  -n envoy-gateway-system \
  --create-namespace
```

This creates the `envoy-gateway-system` namespace.

**Verify the pod is running:**

```bash
kubectl get pods -n envoy-gateway-system

# Expected output:
# NAME                              READY   STATUS    RESTARTS   AGE
# envoy-gateway-64d8866b44-xvc7t    1/1     Running   0          38s
```

**Check CRDs created:**

```bash
kubectl get crd -n envoy-gateway-system
```

You'll see all the Gateway API CRDs registered, including:
- `gatewayclasses.gateway.networking.k8s.io`
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `httproutefilters.gateway.envoyproxy.io`
- `securitypolicies.gateway.envoyproxy.io`
- ...and many more

**Check the logs (it'll be waiting — no GatewayClass accepted yet):**

```bash
kubectl logs envoy-gateway-64d8866b44-xvc7t -n envoy-gateway-system

# You'll see:
# no accepted gatewayclass {"runner": "provider"}
# reconciling gateways...
```

This is fine. The controller is just sitting there, refreshing like a browser before a ticket sale. It needs a GatewayClass to be defined.

> **Platform Engineer work ends here.** DevOps/App Engineer takes over.

---

### Step 2: Deploy the Application

Before exposing anything, we need an app to expose. The following files set up a simple echo backend.

#### `01-SA.yaml` — Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
```

#### `02-DEPLOY.yaml` — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      serviceAccountName: backend
      containers:
        - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
          imagePullPolicy: IfNotPresent
          name: backend
          ports:
            - containerPort: 3000
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
```

#### `03-SVC.yaml` — Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
    service: backend
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: backend
```

---

### Step 3: Configure Gateway API Resources

#### `04-GATEWAY-CLASS.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

> `spec.controllerName` tells Kubernetes which controller implementation to use — in this case, Envoy Proxy.

---

#### `05-GATEWAY.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
spec:
  gatewayClassName: eg       # References our GatewayClass
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

> `gatewayClassName: eg` wires this Gateway to the Envoy controller we defined above.

---

#### `06-HTTP-ROUTE.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend
spec:
  parentRefs:
    - name: eg                 # Attach to our Gateway
  hostnames:
    - "www.example.com"        # Only accept requests for this hostname
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: backend
          port: 3000
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```

> This is the most important resource. It defines:
> - **Which Gateway** to attach to (`parentRefs`)
> - **Which hostname** to accept (`hostnames`) — acts as a virtual host filter
> - **Which backend service** to forward traffic to (`backendRefs`)
> - **Which paths** to match (`matches`)

---

### Step 4: Expose the Service Locally

On cloud (GCP/AWS), the Gateway automatically provisions a cloud load balancer.

On local clusters (KIND, Minikube), you need `port-forward`:

```bash
# Find the Envoy service name
kubectl get svc -n envoy-gateway-system

# Port-forward to local port 9090
kubectl port-forward svc/<envoy-svc-name> 9090:80 \
  -n envoy-gateway-system \
  --address 0.0.0.0
```

The terminal session is now occupied. **Open a new terminal tab.**

---

### Step 5: Test the Setup

```bash
curl --verbose \
  --header "Host: www.example.com" \
  http://localhost:9090/get
```

> We pass the `Host` header manually because we're testing locally. In production, the browser sends your domain automatically.

---

## Feature: URL Rewrite

**Scenario:** Incoming requests arrive at `/get/...` but your backend service expects `/replace/...`. The Gateway rewrites the path transparently.

#### `rewrite-httproute.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-filter-url-rewrite
spec:
  parentRefs:
    - name: eg
  hostnames:
    - path.rewrite.example
  rules:
    - matches:
        - path:
            value: "/get"
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /replace
      backendRefs:
        - name: backend
          port: 3000
```

**How it works:**
- Client sends: `origin/get/path`
- Gateway rewrites to: `origin/replace/path`
- Backend receives the rewritten URL — none the wiser

**Test it:**

```bash
curl -L -vvv \
  --header "Host: path.rewrite.example" \
  "http://localhost:9090/get/origin/path/extra"
```

Check the `path` field in the response — it should show `/replace/origin/path/extra`.

---

## Feature: Traffic Splitting (50/50)

**Situation:** You've deployed a new version of your app. You're not fully confident in it (and you shouldn't be — none of us are).

**Task:** Run both versions simultaneously and split traffic between them.

**Action:** Deploy a second backend and configure HTTPRoute with two `backendRefs` and no explicit weight — Gateway defaults to 50/50.

**Result:** You get to catch production bugs before they catch you.

### Deploy Backend v2

#### `sa-2.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-2
```

#### `deploy-2.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend-2
      version: v1
  template:
    metadata:
      labels:
        app: backend-2
        version: v1
    spec:
      serviceAccountName: backend-2
      containers:
        - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
          imagePullPolicy: IfNotPresent
          name: backend-2
          ports:
            - containerPort: 3000
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
```

#### `svc-2.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-2
  labels:
    app: backend-2
    service: backend-2
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: backend-2
```

### 50/50 HTTPRoute

#### `httproute_traffic_splitting.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-headers
spec:
  parentRefs:
    - name: eg
  hostnames:
    - backends.example
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: backend
          port: 3000
        - group: ""
          kind: Service
          name: backend-2
          port: 3000
```

> No `weight` specified → Gateway splits evenly (50/50).

**Test:**

```bash
curl --header "Host: backends.example" "http://localhost:9090/get"
```

Hit it multiple times — you'll see different `pod` values in the response, confirming traffic is going to both backends.

> ⚠️ **Note:** This is NOT round-robin. Distribution is probabilistic, not deterministic.

---

## Feature: Weight-Based Traffic Splitting (Canary Deployment)

**Situation:** New version deployed. You want to give it *just a little* traffic — say 20% — while keeping 80% on the proven version.

**Task:** Control the traffic ratio using the `weight` field in `backendRefs`.

**Action:** Add `weight` values. They're relative — `8` and `2` means 80% / 20%.

**Result:** Gradual rollout with the ability to instantly shift to 100% new when confidence is earned.

#### `httproute_weighted.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-headers
spec:
  parentRefs:
    - name: eg
  hostnames:
    - backends.example
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: backend        # v1 — trusted
          port: 3000
          weight: 8            # 80% of traffic
        - group: ""
          kind: Service
          name: backend-2      # v2 — canary
          port: 3000
          weight: 2            # 20% of traffic
```

> Applying this updates the existing HTTPRoute — it doesn't create a new one (same `name: http-headers`).

**Test:**

```bash
curl --header "Host: backends.example" "http://localhost:9090/get"
```

Run it ~10 times — roughly 8 responses come from `backend`, 2 from `backend-2`.

When you're confident in `backend-2`, set its weight to `10` and remove `backend`. 100% traffic migrated. No drama. (Hopefully.)

---

## Summary: Ingress vs Gateway API

| Feature | Ingress | Gateway API |
|---|---|---|
| Rate Limiting | ❌ (annotation hack) | ✅ Native |
| WAF Support | ❌ | ✅ |
| Canary Deployment | ❌ (vendor-specific) | ✅ |
| Traffic Splitting | ❌ | ✅ |
| URL Rewrite | ❌ | ✅ |
| Access Separation | ❌ (one file, everyone fights) | ✅ (GatewayClass / HTTPRoute split) |
| Vendor Lock-in | ✅ (annotations are vendor-specific) | ❌ (standardized CRDs) |
| Is it a Custom Resource | ❌ | ✅ (even more extensible) |

---

## Quick Reference: Resource Ownership

```
Platform Engineer
    └── GatewayClass  →  Which controller implementation
    └── Gateway       →  Listeners (protocol/port)

DevOps / App Engineer
    └── HTTPRoute     →  Routing rules, backends, rewrites, weights
```

---

*These notes cover: Ingress limitations, Gateway API motivation, 3-resource architecture, Envoy Gateway setup, basic routing, URL rewrite, traffic splitting, and canary deployments.*
