# Kyverno policy requirements — ingress-nginx → F5 NGINX Ingress (PoC)

**Audience:** Developers implementing policies; **attachment:** examples below for stakeholder confirmation.  
**Related:** `POC-ingress-nginx-migration-kyverno.md`  
**Last updated:** 2026-04-21  

---



---

## 2. Usecase example pattern

---

### Pattern A — 1:1 annotation mapping 

**Intent:** Copy or map one `nginx.ingress.kubernetes.io/...` value to one `nginx.org/...` annotation. *(Exact key names: confirm vs [migration guide](https://kubernetes.nginx.org/ingress-nginx-migration.html) and NIC version.)*

**Before (attach):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-a
  namespace: app-demo
  labels:
    migration.nginx.org/poc: "enabled"
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-svc
                port:
                  number: 80
```

**After (attach — expected patch on same Ingress):**

```yaml
metadata:
  annotations:
    nginx.org/redirect-to-https: "true"   
```



---

### Pattern B — Annotation → controller ConfigMap → `**generate**`

**Intent:** When `Ingress` has e.g. `proxy-body-size`, the **NIC controller ConfigMap** gets the mapped key/value. *(ConfigMap name, namespace, and **data key** must match your NIC install — dev confirms.)*

**Before (attach — excerpt):**

```yaml
metadata:
  name: demo-b
  namespace: app-demo
  labels:
    migration.nginx.org/poc: "enabled"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
```

**After (attach — controller ConfigMap fragment):**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration          # EXAMPLE — use real name from your NIC namespace
  namespace: nginx-ingress           # EXAMPLE
data:
  client-max-body-size: "10m"        # EXAMPLE key — confirm in the docs
```



---

### Pattern C — Basic auth annotations → `**Policy` + `mutate**` (`generate` + `mutate`)

**Intent:** Ingress-nginx basic auth annotations become a `**Policy`** (`spec.basicAuth`) and the Ingress references it via `**nginx.org/policies**`. See [authentication-basic](https://kubernetes.nginx.org/ingress-nginx-migration.html#authentication-basic).

**Before (attach):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-c
  namespace: app-demo
  labels:
    migration.nginx.org/poc: "enabled"
  annotations:
    nginx.ingress.kubernetes.io/auth-type: "basic"
    nginx.ingress.kubernetes.io/auth-secret: "basic-auth-secret"
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  ingressClassName: nginx
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-svc
                port:
                  number: 80
```

**After (attach — two objects):**

```yaml
# 1) Generated Policy
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: demo-c-basic-auth
  namespace: app-demo
spec:
  basicAuth:
    secret: basic-auth-secret
    realm: "Authentication Required"
---
# 2) Same Ingress with reference ( add annotation)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-c
  namespace: app-demo
  annotations:
    nginx.org/policies: "demo-c-basic-auth"

```

**Example Kyverno logic:**

1. `**generate`** — create `Policy` `demo-c-basic-auth` from the Ingress name + `auth-secret` / `auth-realm`.
2. `**mutate**` — set `metadata.annotations["nginx.org/policies"]` to that policy name.

---

### Pattern D — CORS annotations → `**Policy` (`cors`) + `mutate**` (`generate` + `mutate`)

**Intent:** ingress-nginx CORS annotations become a `**Policy`** with `spec.cors` and `**nginx.org/policies**` on the Ingress. See [CORS](https://kubernetes.nginx.org/ingress-nginx-migration.html#cors).

**Before (attach):**

```yaml
metadata:
  name: demo-d
  namespace: app-demo
  labels:
    migration.nginx.org/poc: "enabled"
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://frontend.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
```

**After (attach):**

```yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: demo-d-cors
  namespace: app-demo
spec:
  cors:
    allowOrigin:
      - "https://frontend.example.com"
    allowMethods:
      - GET
      - POST
      - OPTIONS
    allowHeaders:
      - Authorization
      - Content-Type
    allowCredentials: true
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-d
  namespace: app-demo
  annotations:
    nginx.org/policies: "demo-d-cors"
```

**Example Kyverno logic:**

- `**generate`** — build `Policy` `demo-d-cors` from CORS annotations (split comma-separated strings into arrays where the CRD expects lists).
- `**mutate**` — set `nginx.org/policies: demo-d-cors`.

