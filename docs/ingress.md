# Ingress Configuration for IAP

The IAP Helm chart provides comprehensive ingress configuration options to enable both load-balanced access and direct pod access for your Itential Automation Platform deployment.

## Overview

The ingress configuration supports two primary access patterns:

1. **Load Balancer Access**: Routes traffic to all IAP pods through a single hostname
2. **Direct Access**: Provides individual hostnames for each IAP pod instance

## Load Balancer Access

Load balancer access distributes incoming requests across all available IAP pods, providing high availability and load distribution.

### Configuration

```yaml
ingress:
  enabled: true
  loadBalancer:
    enabled: true
    host: "iap.example.com"
    path: "/"
```

### Features

- Automatic load distribution across all pods
- Single entry point for the application
- Supports WebSocket connections (when `useWebSockets` is enabled)
- Integrates with AWS ALB or other ingress controllers

### Load Balancer Options

The chart supports multiple ingress controllers. Choose the one that best fits your environment.

#### AWS Application Load Balancer (ALB)

This load balancing solution uses AWS Application Load Balancer through the AWS Load Balancer Controller.

**ALB Configuration Example:**

```yaml
ingress:
  enabled: true
  className: "alb"
  loadBalancer:
    enabled: true
    host: "iap.example.com"
  annotations:
    alb.ingress.kubernetes.io/backend-protocol: "HTTPS"
    alb.ingress.kubernetes.io/healthcheck-path: "/health/status?exclude-services=true"
    alb.ingress.kubernetes.io/healthcheck-port: "3443"
    alb.ingress.kubernetes.io/healthcheck-protocol: "HTTPS"
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    # Include port 8080 if using Gateway Manager with IAG5, otherwise remove it
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443},{"HTTPS": 8080}]'
    alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=60
    alb.ingress.kubernetes.io/load-balancer-name: "itential-example-lb"
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/success-codes: "200"
    alb.ingress.kubernetes.io/target-type: "ip"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
    alb.ingress.kubernetes.io/websocket-paths: "/ws"
    alb.ingress.kubernetes.io/target-group-attributes: stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=3600
```

**SSL/TLS Certificate Configuration:**

For SSL/TLS termination at the ALB level, specify an AWS Certificate Manager (ACM) certificate using the `certificate-arn` annotation:

```yaml
annotations:
  alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:region:account-id:certificate/certificate-id"
```

The `certificate-arn` annotation:
- Specifies the ARN (Amazon Resource Name) of an ACM certificate
- Enables SSL/TLS termination at the load balancer
- Supports multiple certificates by providing comma-separated ARNs
- The certificate must be in the same AWS region as the ALB
- Requires the ALB to have HTTPS listeners configured (via `listen-ports`)

**Example with certificate:**

```yaml
ingress:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"
```

#### GKE HTTP(S) Load Balancer - GKE Option

On Google Kubernetes Engine, the native ingress controller provisions a Google Cloud HTTP(S) Load Balancer. Because IAP terminates TLS at the pod (port 443), GKE requires a `BackendConfig` custom resource to configure HTTPS backend health checks and session affinity, and a `cloud.google.com/app-protocols` annotation on the Service to tell the load balancer to use HTTPS when communicating with pods.

> **Note:** The `BackendConfig` and `ManagedCertificate` resources must be created separately — they are not rendered by this chart. Apply them to your cluster before or alongside the Helm release.

**BackendConfig (apply separately):**

```yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: iap-backend-config
spec:
  healthCheck:
    checkIntervalSec: 15
    timeoutSec: 5
    healthyThreshold: 2
    unhealthyThreshold: 2
    type: HTTPS
    requestPath: /health/status
    port: 3443
  sessionAffinity:
    affinityType: GENERATED_COOKIE
    affinityCookieTtlSec: 3600
  timeoutSec: 60
  connectionDraining:
    drainingTimeoutSec: 60
```

**ManagedCertificate (apply separately):**

```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: iap-managed-cert
spec:
  domains:
    - iap.example.com
    - iap-prod-0.example.com
    - iap-prod-1.example.com
```

**Helm values configuration:**

```yaml
service:
  type: ClusterIP
  name: iap-service
  port: 443
  annotations:
    # Tell GKE LB to use HTTPS when communicating with pods
    cloud.google.com/app-protocols: '{"https":"HTTPS"}'
    # Enable container-native load balancing (pod-level routing, equivalent to ALB target-type: ip)
    cloud.google.com/neg: '{"ingress": true}'
    # Reference the BackendConfig for health checks and session affinity
    cloud.google.com/backend-config: '{"default": "iap-backend-config"}'

ingress:
  enabled: true
  className: "gce"
  loadBalancer:
    enabled: true
    host: iap.example.com
    path: /
  directAccess:
    enabled: true
    baseDomain: example.com
    path: /
  annotations:
    # Reference the ManagedCertificate for SSL
    networking.gke.io/managed-certificates: "iap-managed-cert"
    # Use a static external IP (reserve one in GCP Console first)
    kubernetes.io/ingress.global-static-ip-name: "iap-static-ip"
    # Force HTTPS
    kubernetes.io/ingress.allow-http: "false"
    external-dns.alpha.kubernetes.io/hostname: iap.example.com
    external-dns.alpha.kubernetes.io/ttl: "300"
  # TLS block is not used with GKE ManagedCertificates — leave empty
  tls: []
```

> **WebSocket support:** GKE HTTP(S) LB supports WebSockets natively. No additional annotation is required. Ensure port 8080 is exposed via a separate Service or NodePort if using IAG5/Gateway Manager.

---

#### Azure Application Gateway Ingress Controller (AGIC) - AKS Option

On Azure Kubernetes Service, the Application Gateway Ingress Controller (AGIC) provisions an Azure Application Gateway. AGIC uses annotation-based configuration similar to NGINX but with `appgw.ingress.kubernetes.io/` prefixed annotations.

Because IAP terminates TLS at the pod (port 443), the `backend-protocol: "https"` annotation is required. SSL certificates are attached to the Application Gateway frontend, not managed through Kubernetes secrets.

**Helm values configuration:**

```yaml
service:
  type: ClusterIP
  name: iap-service
  port: 443

ingress:
  enabled: true
  className: "azure-application-gateway"
  loadBalancer:
    enabled: true
    host: iap.example.com
    path: /
  directAccess:
    enabled: true
    baseDomain: example.com
    path: /
  annotations:
    # Use HTTPS when communicating with backend pods (TLS terminates at the pod)
    appgw.ingress.kubernetes.io/backend-protocol: "https"
    # Force HTTPS redirect on the frontend
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    # Reference an SSL certificate pre-uploaded to Application Gateway
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: "iap-ssl-cert"
    # Health probe configuration
    appgw.ingress.kubernetes.io/health-probe-path: "/health/status"
    appgw.ingress.kubernetes.io/health-probe-port: "3443"
    appgw.ingress.kubernetes.io/health-probe-status-codes: "200-399"
    appgw.ingress.kubernetes.io/health-probe-interval: "15"
    appgw.ingress.kubernetes.io/health-probe-timeout: "5"
    appgw.ingress.kubernetes.io/health-probe-unhealthy-threshold: "2"
    # Session affinity (equivalent to ALB stickiness)
    appgw.ingress.kubernetes.io/cookie-based-affinity: "Enabled"
    appgw.ingress.kubernetes.io/cookie-based-affinity-distinct-name: "true"
    # Request and connection timeouts
    appgw.ingress.kubernetes.io/request-timeout: "60"
    appgw.ingress.kubernetes.io/connection-draining: "true"
    appgw.ingress.kubernetes.io/connection-draining-timeout: "30"
    external-dns.alpha.kubernetes.io/hostname: iap.example.com
    external-dns.alpha.kubernetes.io/ttl: "300"
  # TLS block is not used with AGIC frontend certificates — leave empty
  tls: []
```

> **SSL Certificate:** The `appgw-ssl-certificate` annotation references a certificate by name that must already be uploaded to the Application Gateway in Azure. This is separate from Kubernetes secrets or cert-manager. Alternatively, you can integrate with Azure Key Vault via the AGIC add-on.

> **WebSocket support:** Azure Application Gateway supports WebSockets natively. Ensure port 8080 is included in the Application Gateway listener configuration if using IAG5/Gateway Manager.

---

#### Traefik - Bare-Metal / On-Premises Option

Traefik is a cloud-native ingress controller well suited to bare-metal and on-premises Kubernetes clusters. Because IAP terminates TLS at the pod (port 443), Traefik must be configured to re-encrypt the backend connection — it terminates TLS from the client, then reconnects to the IAP pod over HTTPS.

**Step 1 — Install Traefik:**

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=false \
  --set "additionalArguments[0]=--serversTransport.insecureSkipVerify=true"
```

The `--serversTransport.insecureSkipVerify=true` flag tells Traefik to skip TLS certificate verification when connecting to backend pods. This is required because IAP pods use self-signed certificates that do not include pod IP SANs.

> **Note:** In Traefik v3, the `ServersTransport` CRD annotation (`traefik.ingress.kubernetes.io/service.serversTransport`) on standard Kubernetes Ingress objects does not reliably apply per-service backend TLS settings. The global `--serversTransport.insecureSkipVerify=true` flag is the recommended approach for IAP deployments.

**TLS certificate considerations:**

`insecureSkipVerify` only affects the **backend connection** (Traefik → IAP pod). It has no effect on the **frontend connection** (client browser → Traefik), which uses the certificate from `iap-tls-secret`. Clients only ever see the frontend certificate.

| Connection | Certificate | Verified by |
|---|---|---|
| Client → Traefik | `iap-tls-secret` (customer-provided or CA-signed) | Client browser |
| Traefik → IAP pod | Self-signed (pod-level) | Skipped via `insecureSkipVerify` |

This is a common and accepted pattern. If a customer provides a CA-signed certificate that includes pod DNS names as SANs, `insecureSkipVerify` can be removed. However, pod IPs and their derived DNS names (e.g., `10-200-2-131.na.pod.cluster.local`) are ephemeral — they change on every pod restart — making it impractical to maintain a static certificate for them. The recommended approach is to keep `insecureSkipVerify=true` for the backend and ensure the frontend certificate presented to clients is properly signed.

**Step 2 — Apply the ServersTransport CRD** (namespace-scoped, required for Traefik to identify the backend transport configuration):

```bash
kubectl apply -f - <<EOF
apiVersion: traefik.io/v1alpha1
kind: ServersTransport
metadata:
  name: iap-servers-transport
  namespace: <your-namespace>
spec:
  insecureSkipVerify: true
EOF
```

**Step 3 — Helm values configuration:**

```yaml
ingress:
  enabled: true
  className: "traefik"
  loadBalancer:
    enabled: true
    host: iap.example.com
    path: /
  directAccess:
    enabled: true
    baseDomain: example.com
    path: /
  annotations:
    # Route incoming traffic through the HTTPS (websecure) entrypoint
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # Enable TLS on this router
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Reference the ServersTransport CRD — format: <namespace>-<name>@kubernetescrd
    # If deploying to the default namespace, use: default-iap-servers-transport@kubernetescrd
    traefik.ingress.kubernetes.io/service.serversTransport: <namespace>-iap-servers-transport@kubernetescrd
  tls:
  - hosts:
    - iap.example.com
    secretName: iap-tls-secret
```

> **Cloud environments:** On EKS, GKE, or AKS, Traefik's `LoadBalancer` service is automatically assigned an external IP by the cloud provider. No additional configuration is needed.

> **Bare-metal environments:** Without a cloud provider, the `LoadBalancer` service will remain in `<pending>` state. Install [MetalLB](https://metallb.universe.tf/) to assign external IPs, or consult your cluster administrator for how external traffic is routed to the cluster.

> **WebSocket support:** Traefik supports WebSockets natively. No additional annotation is required.

---

#### HAProxy - Bare-Metal / On-Premises Option

The HAProxy Kubernetes Ingress Controller is a high-performance, enterprise-grade ingress controller well suited to bare-metal and on-premises clusters. Because IAP terminates TLS at the pod (port 443), HAProxy must be configured to re-encrypt the backend connection — it terminates TLS from the client, then reconnects to the IAP pod over HTTPS.

Unlike Traefik, HAProxy does not require a global startup flag or a separate CRD prerequisite. Backend TLS behavior is controlled entirely through annotations on the Ingress object.

**Step 1 — Install HAProxy Kubernetes Ingress Controller:**

```bash
helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update
helm install haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-controller \
  --create-namespace \
  --set controller.ingressClass=haproxy \
  --set controller.ingressClassResource.enabled=true \
  --set controller.ingressClassResource.name=haproxy \
  --set controller.ingressClassResource.isDefaultClass=false
```

**TLS certificate considerations:**

`haproxy.org/server-ssl-verify: "none"` only affects the **backend connection** (HAProxy → IAP pod). It has no effect on the **frontend connection** (client browser → HAProxy), which uses the certificate from `iap-tls-secret`. Clients only ever see the frontend certificate.

| Connection | Certificate | Verified by |
|---|---|---|
| Client → HAProxy | `iap-tls-secret` (customer-provided or CA-signed) | Client browser |
| HAProxy → IAP pod | Self-signed (pod-level) | Skipped via `server-ssl-verify: none` |

This is a common and accepted pattern. Pod IPs and their derived DNS names are ephemeral — they change on every pod restart — making it impractical to maintain a static certificate for backend pods. The recommended approach is to skip verification for the backend and ensure the frontend certificate presented to clients is properly signed.

**Step 2 — Helm values configuration:**

```yaml
ingress:
  enabled: true
  className: "haproxy"
  loadBalancer:
    enabled: true
    host: iap.example.com
    path: /
  directAccess:
    enabled: true
    baseDomain: example.com
    path: /
  annotations:
    # Re-encrypt to backend IAP pods over HTTPS (IAP terminates TLS at the pod)
    haproxy.org/server-ssl: "true"
    # Skip backend certificate verification — IAP pods use self-signed certs without pod IP SANs
    haproxy.org/server-ssl-verify: "none"
    # Redirect HTTP to HTTPS
    haproxy.org/ssl-redirect: "true"
    # Cookie-based session affinity — required for IAP UI actions that must reach the same pod
    haproxy.org/cookie-persistence: "iap-server"
    # Connection and request timeouts
    haproxy.org/timeout-connect: "5s"
    haproxy.org/timeout-client: "300s"
    haproxy.org/timeout-server: "300s"
    # WebSocket tunnel timeout — keep alive for long-lived WebSocket connections
    haproxy.org/timeout-tunnel: "3600s"
    # Health check path for backend IAP pods
    haproxy.org/check: "true"
    haproxy.org/check-http: "/health/status?exclude-services=true"
  tls:
  - hosts:
    - iap.example.com
    secretName: iap-tls-secret
```

> **WebSocket support (IAG5/Gateway Manager):** When `useWebSockets: true` is set, the IAP chart renders a single Ingress with two backends: `/` → port 443 (HTTPS) and `/ws` → port 8080 (plain WebSocket). HAProxy applies `haproxy.org/server-ssl: "true"` to every backend in the Ingress object — there is no per-path SSL override. This causes HAProxy to attempt an SSL handshake to port 8080, which fails. The fix is a second Ingress for `/ws` with `haproxy.org/server-ssl: "false"`. See the [WebSocket SSL Conflict](#why-this-matters-for-websocket-iag5gateway-manager) section below for the full example.

> **Cloud environments:** On EKS, GKE, or AKS, HAProxy's `LoadBalancer` service is automatically assigned an external IP by the cloud provider. No additional configuration is needed.

> **Bare-metal environments:** Without a cloud provider, the `LoadBalancer` service will remain in `<pending>` state. Install [MetalLB](https://metallb.universe.tf/) to assign external IPs, or consult your cluster administrator for how external traffic is routed to the cluster.

---

#### Contour (Envoy) - Bare-Metal / On-Premises Option

Contour is a CNCF-graduated ingress controller that uses Envoy as its data plane. It is well suited to bare-metal and on-premises clusters and works with the standard `networking.k8s.io/v1` Ingress resource — no chart changes required. Because IAP terminates TLS at the pod (port 443), Contour must re-encrypt the backend connection.

Contour's key advantage over HAProxy for IAP is how it scopes backend TLS: the `projectcontour.io/upstream-protocol.tls` annotation is placed on the **Service** and lists only the ports that should use TLS. Port 8080 (WebSocket) is not listed, so Contour routes it as plain HTTP — the WebSocket SSL conflict present in HAProxy does not occur.

> **Session affinity:** Cookie-based sticky sessions are not supported for standard Kubernetes Ingress resources in Contour. Session affinity requires Contour's `HTTPProxy` CRD. For testing, IAP's own session tokens function across pods. Production deployments should evaluate `HTTPProxy` if same-pod routing is required.

**Step 1 — Install Contour:**

```bash
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

> **Bare-metal environments:** Without a cloud provider, the Envoy service will remain in `<pending>` state. Patch it to NodePort and retrieve the assigned port for wiring to your external load balancer:
> ```bash
> kubectl patch svc envoy -n projectcontour -p '{"spec":{"type":"NodePort"}}'
> kubectl get svc envoy -n projectcontour \
>   -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'
> ```

**TLS certificate considerations:**

The `projectcontour.io/upstream-protocol.tls` annotation on the Service tells Contour's Envoy to use TLS only for the listed ports. For standard Ingress resources, Envoy does not validate backend certificates when no CA is explicitly configured — self-signed pod certificates work without additional configuration.

| Connection | Certificate | Verified by |
|---|---|---|
| Client → Contour | `iap-tls-secret` (customer-provided or CA-signed) | Client browser |
| Contour → IAP pod (port 443) | Self-signed (pod-level) | Not verified (no CA configured) |
| Contour → IAP pod (port 8080) | None — plain HTTP | N/A |

**Step 2 — Helm values configuration:**

```yaml
service:
  type: ClusterIP
  name: iap-service
  port: 443
  annotations:
    # Tell Contour/Envoy to use TLS only for port 443.
    # Port 8080 (WebSocket) is intentionally excluded — it uses plain HTTP.
    projectcontour.io/upstream-protocol.tls: "443"

ingress:
  enabled: true
  className: "contour"
  loadBalancer:
    enabled: true
    host: iap.example.com
    path: /
  directAccess:
    enabled: true
    baseDomain: example.com
    hostOverride: "iap-{ns}-contour"
    path: /
  annotations:
    # Redirect HTTP to HTTPS
    ingress.kubernetes.io/force-ssl-redirect: "true"
    # Response timeout — set high for long-lived WebSocket connections
    projectcontour.io/response-timeout: "3600s"
  tls:
  - hosts:
    - iap.example.com
    secretName: iap-tls-secret
```

> **Cloud environments:** On EKS, GKE, or AKS, Contour's Envoy `LoadBalancer` service is automatically assigned an external IP by the cloud provider. No additional configuration is needed.

> **Bare-metal environments:** Without a cloud provider, the `LoadBalancer` service will remain in `<pending>` state. Install [MetalLB](https://metallb.universe.tf/) to assign external IPs, or consult your cluster administrator for how external traffic is routed to the cluster.

> **WebSocket support:** Contour supports WebSockets natively. No annotation is required. The `projectcontour.io/response-timeout` annotation controls the idle timeout for long-lived connections including WebSocket.

---

### Backend SSL Behavior by Controller

IAP pods terminate TLS internally at port 3443 (self-signed certificates). Every ingress controller must re-encrypt the backend connection — it terminates TLS from the client and then reconnects to the IAP pod over HTTPS. How each controller handles this, and how granular that configuration is, differs significantly.

| Controller | How Backend SSL is Configured | Granularity |
|---|---|---|
| **ALB** | `alb.ingress.kubernetes.io/backend-protocol: HTTPS` annotation | Per-Ingress |
| **GKE** | `cloud.google.com/app-protocols` on the Service | Per-Service port |
| **Azure AGIC** | `appgw.ingress.kubernetes.io/backend-protocol: https` annotation | Per-Ingress |
| **Traefik** | Global `--serversTransport.insecureSkipVerify=true` startup flag | Global (all backends) |
| **HAProxy** | `haproxy.org/server-ssl: "true"` annotation | Per-Ingress object (applies to all backends in the Ingress) |
| **Contour** | `projectcontour.io/upstream-protocol.tls: "<ports>"` on the Service | Per-Service port |

**Why this matters for WebSocket (IAG5/Gateway Manager):**

When `useWebSockets: true` is set, the IAP chart renders a single Ingress with two backends:

- `/` → `iap-service:443` — speaks HTTPS, needs `server-ssl: true`
- `/ws` → `iap-service:8080` — speaks plain WebSocket (HTTP), must NOT use SSL

**Traefik** avoids this conflict because backend SSL is a global setting applied at the controller level — it re-encrypts port 443 but routes port 8080 as plain HTTP based on the path, without SSL. No per-path configuration is needed.

**HAProxy** applies `haproxy.org/server-ssl: "true"` to every backend in the Ingress object — there is no per-path SSL override. With both `/` and `/ws` in the same Ingress, HAProxy attempts an SSL handshake to port 8080, which fails with `SSL handshake failure (Connection refused)` and marks the WebSocket backend as DOWN.

**Fix: Separate Ingress for the WebSocket path**

Create a second Ingress for `/ws` without `server-ssl: true`. HAProxy will route `/ws` to port 8080 as plain HTTP while the main Ingress continues to use SSL for port 443.

**NA environment example** (`iap-na-k8s.pe.itential.io`):

Apply this manually alongside the Helm-rendered Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: iap-ingress-ws
  namespace: na
  annotations:
    kubernetes.io/description: "IAP WebSocket ingress for Gateway Manager (IAG5). Plain HTTP — no server-ssl."
    haproxy.org/server-ssl: "false"
    haproxy.org/timeout-connect: "5s"
    haproxy.org/timeout-client: "300s"
    haproxy.org/timeout-server: "300s"
    haproxy.org/timeout-tunnel: "3600s"
    haproxy.org/cookie-persistence: "iap-server"
spec:
  ingressClassName: haproxy
  tls:
  - hosts:
    - iap-na-k8s.pe.itential.io
    secretName: iap-tls-secret
  rules:
  - host: iap-na-k8s.pe.itential.io
    http:
      paths:
      - backend:
          service:
            name: iap-service
            port:
              number: 8080
        path: /ws
        pathType: Prefix
```

> **Note:** Do not add `haproxy.org/check` or `haproxy.org/check-http` to the WebSocket Ingress. Health checks over port 8080 use a different protocol than the IAP health endpoint (port 3443). HAProxy will verify backend availability through the WebSocket connection itself.

---

#### Load Balancer Comparison

> **Note:** Ingress NGINX is not included below. Kubernetes SIG Network has announced its retirement — best-effort maintenance ended in March 2026, with no further releases or security fixes. Existing deployments will continue to function, but new deployments should use one of the supported options below.

| Feature | ALB | GKE HTTP(S) LB | Azure AGIC | Traefik | HAProxy | Contour |
|---------|-----|----------------|------------|---------|---------|---------|
| **Provider** | AWS Native | GCP Native | Azure Native | Self-hosted | Self-hosted | Self-hosted |
| **Backend HTTPS** | Annotation | Service annotation + BackendConfig | Annotation | Global flag | Annotation | Service annotation |
| **Backend SSL Granularity** | Per-Ingress | Per-Service port | Per-Ingress | Global (all backends) | Per-Ingress object | Per-Service port |
| **SSL Termination** | At load balancer | At load balancer | At load balancer | At ingress (re-encrypt) | At ingress (re-encrypt) | At ingress (re-encrypt) |
| **WebSocket + SSL mix** | Supported natively | Supported natively | Supported natively | Supported natively | Requires separate Ingress for `/ws` | Supported natively |
| **WebSocket Support** | Native | Native | Native | Native | Native (tunnel timeout annotation) | Native (response-timeout annotation) |
| **Session Affinity** | Target group level | BackendConfig (cookie) | Annotation (cookie) | Middleware (sticky sessions) | Annotation (cookie) | HTTPProxy CRD only |
| **Health Checks** | Annotations | BackendConfig CRD | Annotations | Passive (via response codes) | Annotations | Passive (Envoy) |
| **Pod-Level Routing** | `target-type: ip` | NEG (`cloud.google.com/neg`) | Default | Default | Default | Default |
| **Prerequisite CRD** | None | BackendConfig (out-of-band) | None | ServersTransport CRD | None | None |
| **Best For** | AWS EKS | GKE | AKS | Bare-metal / on-prem | Bare-metal / on-prem | Bare-metal / on-prem |

## Direct Access

Direct access allows you to connect to specific IAP pod instances. This is particularly useful for:

- Debugging specific pod instances
- Administrative tasks that require direct pod access
- UI actions that need direct pod connectivity
- Development and troubleshooting scenarios

### Default Hostname Generation

By default, direct access hostnames are generated using this pattern:

```
{iap.name}-{namespace}-{pod-index}.{baseDomain}
```

**Example Configuration:**
```yaml
ingress:
  directAccess:
    enabled: true
    baseDomain: "example.com"
```

**Generated Hostnames** (with `replicaCount: 3`, namespace `production`):
- `iap-production-0.example.com`
- `iap-production-1.example.com` 
- `iap-production-2.example.com`

### Custom Hostname Override

Use the `hostOverride` setting to customize the hostname prefix:

```yaml
ingress:
  directAccess:
    enabled: true
    baseDomain: "example.com"
    hostOverride: "iap-prod"
```

**Generated Hostnames** (with `replicaCount: 3`):
- `iap-prod-0.example.com`
- `iap-prod-1.example.com`
- `iap-prod-2.example.com`

### Configuration Options

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ingress.directAccess.enabled` | bool | `true` | Enable direct access to individual pods |
| `ingress.directAccess.baseDomain` | string | `"example.com"` | Base domain for generating hostnames |
| `ingress.directAccess.hostOverride` | string | `""` | Custom prefix for hostnames (optional) |
| `ingress.directAccess.path` | string | `"/"` | URL path for direct access routes |

## Complete Configuration Example

Here's a comprehensive example combining both access methods:

```yaml
ingress:
  enabled: true
  className: "alb"
  pathType: "Prefix"
  
  # Load balancer configuration
  loadBalancer:
    enabled: true
    host: "iap.your-domain.com"
    path: "/"
  
  # Direct access configuration
  directAccess:
    enabled: true
    baseDomain: "your-domain.com"
    hostOverride: "iap-prod"
    path: "/"
  
  # ALB-specific annotations
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/target-type: "ip"
    alb.ingress.kubernetes.io/backend-protocol: "HTTPS"
    alb.ingress.kubernetes.io/healthcheck-path: "/health/status"
```

## TLS Configuration

Both access methods support TLS termination. Configure certificates using cert-manager or manually:

```yaml
ingress:
  tls:
    - secretName: iap-tls-secret
      hosts:
        - "iap.your-domain.com"
        - "iap-prod-0.your-domain.com"
        - "iap-prod-1.your-domain.com"
        - "iap-prod-2.your-domain.com"
```

## Process Exporter Integration

When `processExporter.enabled` is `true`, direct access routes automatically include metrics endpoints:

- **Metrics Path**: `/metrics`
- **Port**: Process exporter port (default: 9256)
- **Protocol**: HTTP

This allows monitoring systems to scrape metrics from individual pod instances.

## Troubleshooting

### Common Issues

1. **Invalid hostname errors**: Ensure `hostOverride` doesn't contain invalid characters
2. **DNS resolution problems**: Verify your DNS provider supports the generated hostnames
3. **TLS certificate issues**: Make sure certificates cover all generated hostnames

### Validation

Test your configuration with:

```bash
# Generate template to verify hostnames
helm template iap charts/iap/ -f your-values.yaml | grep -A5 "host:"

# Test specific pod access
curl -k https://iap-prod-0.your-domain.com/health/status
```

