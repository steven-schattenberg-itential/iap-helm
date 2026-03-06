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
    traefik.ingress.kubernetes.io/service.serversTransport: <namespace>-iap-servers-transport@kubernetescrd
  tls:
  - hosts:
    - iap.example.com
    secretName: iap-tls-secret
```

**NodePort and external load balancer:**

In bare-metal environments, Traefik's `LoadBalancer` service will remain in `<pending>` state without a cloud provider or MetalLB. Traffic reaches Traefik via NodePort. When the external load balancer routes to a fixed port (e.g., 443 or a custom port), configure Traefik with a matching fixed NodePort to avoid reconfiguring the LB on each install:

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=false \
  --set "additionalArguments[0]=--serversTransport.insecureSkipVerify=true" \
  --set "ports.websecure.nodePort=<your-port>"
```

> **WebSocket support:** Traefik supports WebSockets natively. No additional annotation is required.

---

#### Load Balancer Comparison

> **Note:** Ingress NGINX is not included below. Kubernetes SIG Network has announced its retirement — best-effort maintenance ended in March 2026, with no further releases or security fixes. Existing deployments will continue to function, but new deployments should use one of the supported options below.

| Feature | ALB | GKE HTTP(S) LB | Azure AGIC | Traefik |
|---------|-----|----------------|------------|---------|
| **Provider** | AWS Native | GCP Native | Azure Native | Self-hosted |
| **Backend HTTPS** | Annotation | Service annotation + BackendConfig | Annotation | Global flag |
| **SSL Termination** | At load balancer | At load balancer | At load balancer | At ingress (re-encrypt) |
| **WebSocket Support** | Native | Native | Native | Native |
| **Session Affinity** | Target group level | BackendConfig (cookie) | Annotation (cookie) | Middleware (sticky sessions) |
| **Health Checks** | Annotations | BackendConfig CRD | Annotations | Passive (via response codes) |
| **Pod-Level Routing** | `target-type: ip` | NEG (`cloud.google.com/neg`) | Default | Default |
| **Best For** | AWS EKS | GKE | AKS | Bare-metal / on-prem |

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

