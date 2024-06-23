# Traefik Real IP Plugin

The `traefik_real_ip` plugin for Traefik enhances the ability to extract and set the real client IP address from the `X-Forwarded-For` header. This is particularly useful when Traefik is deployed behind a load balancer or proxy where the actual client IP address can be obscured.

## Features

- Extracts the real client IP address from the `X-Forwarded-For` header.
- Configurable depth (`forwardedForDepth`) for selecting which IP from `X-Forwarded-For` to use as the real IP.
- Sets the `X-Real-Ip` header with the determined real client IP address.
- Mitigates IP spoofing by ensuring the `X-Real-Ip` header reflects the actual client IP (`REAL_IP`).

## Configuration

### Configuration Options

The plugin supports the following configuration option:

| Option             | Description |
| ------------------ | ----------- |
| `forwardedForDepth`| Specifies the depth to look into the `X-Forwarded-For` header. Default is `1`, meaning it uses the last IP unless configured otherwise. |

### Example Configuration

#### Static Configuration

```yaml
pilot:
  token: xxxx

experimental:
  plugins:
    traefik-real-ip:
      modulename: github.com/safeer-qdcorp/traefik-real-ip
      version: main
```

#### Dynamic Configuration (Middleware)

```yaml
http:
  middlewares:
    traefik-real-ip:
      plugin:
        traefik-real-ip:
          forwardedForDepth: 2
```

### Deployment Configuration

#### Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: traefik
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      containers:
        - name: traefik
          image: traefik:v2.4
          args:
            - --api.insecure
            - --entrypoints.web.Address=:80
            - --providers.kubernetescrd
            - --experimental.plugins.traefik-real-ip.modulename=github.com/safeer-qdcorp/traefik-real-ip
            - --experimental.plugins.traefik-real-ip.version=main
          ports:
            - name: web
              containerPort: 80
          resources:
            requests:
              cpu: 300m
            limits:
              cpu: 500m

---

apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: traefik-real-ip
spec:
  plugin:
    traefik-real-ip:
      forwardedForDepth: 2

---

apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingress-example
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - kind: Rule
      match: Host(`example.com`) && PathPrefix(`/`)
      services:
        - name: example-service
          port: 80
      middlewares:
        - name: traefik-real-ip
```

### Configuration Documentation

This plugin ensures that Traefik accurately determines the real client IP address by evaluating the `X-Forwarded-For` header. Adjust the `forwardedForDepth` parameter to suit your environment and security requirements.

### Preventing IP Spoofing

To prevent IP spoofing attacks, configure the `forwardedForDepth` parameter appropriately. For instance, with `forwardedForDepth: 2`, the plugin ensures that the `X-Real-Ip` header always reflects the actual client IP (`REAL_IP`) from the `X-Forwarded-For` header.

### Example Usage

When sending a request with a custom `X-Forwarded-For` header:

```bash
curl -X POST https://example.com/whoami -H "X-Forwarded-For: 10.0.0.1, REAL_IP, CF_IP, LB_IP, PROXY_IP"
```

Assuming `forwardedForDepth: 2`, the resulting headers would include:

```
Hostname: <hostname>
IP: <actual_real_ip>
...
X-Real-Ip: <actual_real_ip>
```

This demonstrates how the plugin accurately sets the `X-Real-Ip` header based on the `X-Forwarded-For` header, ensuring correct identification of the client IP even when Traefik is behind a proxy or load balancer.

---