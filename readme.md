


---

### **Step 1: Install Istio on AKS**

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
export PATH=$PWD/bin:$PATH

# Install Istio with Ingress & Egress Gateways
istioctl install --set profile=default -y
```

Enable **automatic sidecar injection** for your namespace:

```bash
kubectl create namespace demo
kubectl label namespace demo istio-injection=enabled
```

---

### **Step 2: Deploy Sample Services**

(Assume you already have `service-a` and `service-b` YAML manifests without changes.)

Apply them:

```bash
kubectl apply -f service-a.yaml -n demo
kubectl apply -f service-b.yaml -n demo
```

---

### **Step 3: Istio Ingress Gateway for Incoming Traffic**

Create a `Gateway` and `VirtualService`:

#### **Gateway**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: demo-gateway
  namespace: demo
spec:
  selector:
    istio: ingressgateway # use istio's default ingress gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

#### **VirtualService**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: demo-virtualservice
  namespace: demo
spec:
  hosts:
  - "*"
  gateways:
  - demo-gateway
  http:
  - match:
    - uri:
        prefix: /service-a
    rewrite:
      uri: /
    route:
    - destination:
        host: service-a
        port:
          number: 80
  - match:
    - uri:
        prefix: /service-b
    rewrite:
      uri: /
    route:
    - destination:
        host: service-b
        port:
          number: 80
```

**Expose External IP:**

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Access using `http://<EXTERNAL-IP>/service-a`.

---

### **Step 4: Egress Gateway for Outbound Access**

Let’s allow microservices to call `httpbin.org`.

#### **Egress Gateway**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
  namespace: istio-system
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.org"
```

#### **DestinationRule for Egress**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egress-httpbin
  namespace: istio-system
spec:
  host: httpbin.org
  trafficPolicy:
    tls:
      mode: DISABLE
```

#### **VirtualService for Egress**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-httpbin
  namespace: istio-system
spec:
  hosts:
  - "httpbin.org"
  gateways:
  - istio-egressgateway
  - mesh
  tcp:
  - match:
    - port: 80
    route:
    - destination:
        host: httpbin.org
        port:
          number: 80
```

---

### **Step 5: Test**

From `service-a` pod:

```bash
kubectl exec -it <service-a-pod> -n demo -- curl http://httpbin.org/get
```

---

✅ This setup ensures:

* **Ingress**: External traffic → Istio Ingress → Services (`/service-a`, `/service-b`).
* **Egress**: Internal services → Istio Egress Gateway → External API (`httpbin.org`).

---

