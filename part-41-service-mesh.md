# Part 41: Service Mesh - Istio
## ขั้นตอนที่ 1241-1275

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** Service-to-service communication, security, observability

---

## ขั้นตอนที่ 1241: Service Mesh Concepts

```
Service Mesh คืออะไร?
  Layer ที่จัดการ service-to-service communication
  ทำงาน transparent ไม่ต้องแก้ application code

ทำอะไรได้บ้าง?
  ✅ Mutual TLS (mTLS) - encrypt service-to-service
  ✅ Load balancing (advanced: weighted, canary)
  ✅ Circuit breaking
  ✅ Rate limiting
  ✅ Distributed tracing (automatic)
  ✅ Traffic management (canary, blue/green)
  ✅ Service discovery
  ✅ Retries with backoff

Istio Architecture:
  Data Plane:
    Envoy proxy (sidecar) injected into each pod
    Intercepts all network traffic
  
  Control Plane:
    Istiod = Config + Pilot + Citadel
    Manages sidecar config
    Issues certificates for mTLS

Sidecar Injection:
  Each pod gets 2 containers:
    1. Your app container
    2. Envoy proxy (injected automatically)
```

---

## ขั้นตอนที่ 1242: Install Istio

```bash
# Install Istio CLI
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/istio-1.20.0/bin:$PATH

# Install Istio in Kubernetes
istioctl install --set profile=demo -y

# Enable sidecar injection for namespace
kubectl label namespace myapp istio-injection=enabled

# Verify installation
kubectl get pods -n istio-system

# Install addons (Prometheus, Grafana, Jaeger, Kiali)
kubectl apply -f istio-1.20.0/samples/addons/

# Access Kiali dashboard
istioctl dashboard kiali
```

---

## ขั้นตอนที่ 1243: Traffic Management

```yaml
# VirtualService - traffic routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
  namespace: myapp
spec:
  hosts:
    - product-service
  http:
    # Canary: route 10% traffic to v2
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: product-service
            subset: v2
    
    - route:
        - destination:
            host: product-service
            subset: v1
          weight: 90
        - destination:
            host: product-service
            subset: v2
          weight: 10
    
    # Fault injection for testing
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 5s
        abort:
          percentage:
            value: 5
          httpStatus: 503
      route:
        - destination:
            host: product-service
            subset: v1

---
# DestinationRule - load balancing + circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service
  namespace: myapp
spec:
  host: product-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        pendingHttpRequests: 100
    
    # Circuit breaker
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

---

## ขั้นตอนที่ 1244: Mutual TLS

```yaml
# Enable strict mTLS in namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: myapp
spec:
  mtls:
    mode: STRICT  # All service-to-service must use mTLS

---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-authz
  namespace: myapp
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    # Allow only order-service to call product-service
    - from:
        - source:
            principals: ["cluster.local/ns/myapp/sa/order-service"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/products*"]
    
    # Allow admin service full access
    - from:
        - source:
            principals: ["cluster.local/ns/myapp/sa/admin-service"]
```

---

## ขั้นตอนที่ 1245: Retry and Timeout

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - timeout: 10s  # Overall timeout
      
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: gateway-error,connect-failure,retriable-4xx
      
      route:
        - destination:
            host: payment-service
```

---

## ขั้นตอนที่ 1246-1275: Spring Boot with Istio

```java
// Application code stays the same!
// Istio handles:
// - mTLS between services
// - Retries
// - Circuit breaking
// - Distributed tracing (add headers)

// Just configure app for tracing headers propagation
@Component
public class TracingHeaderPropagator implements ClientHttpRequestInterceptor {
    
    @Override
    public ClientHttpResponse intercept(
        HttpRequest request, byte[] body, ClientHttpRequestExecution execution
    ) throws IOException {
        
        // Propagate Istio/Zipkin headers
        ServletRequestAttributes attrs = 
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        
        if (attrs != null) {
            HttpServletRequest req = attrs.getRequest();
            Stream.of("x-request-id", "x-b3-traceid", "x-b3-spanid", 
                      "x-b3-parentspanid", "x-b3-sampled", "x-b3-flags")
                .forEach(header -> {
                    String value = req.getHeader(header);
                    if (value != null) {
                        request.getHeaders().add(header, value);
                    }
                });
        }
        
        return execution.execute(request, body);
    }
}

// Summary: Istio benefits for Spring Boot
// 1. Zero-code TLS between services
// 2. Automatic distributed tracing
// 3. Fine-grained traffic control
// 4. Service-to-service auth
// 5. Canary deployments without code changes
// 6. Circuit breaking at infrastructure level
```

---

*[← Part 40: GraphQL](./part-40-graphql.md) | [Part 42: CI/CD Pipeline →](./part-42-cicd.md)*
