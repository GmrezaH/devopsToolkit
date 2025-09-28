# Traefik Setup

[Traefik](https://traefik.io/) is an open-source edge router that functions as a reverse proxy and load balancer. It dynamically configures routes based on service discovery and supports Docker container integration.

## Preparation

Create a directory for dynamic configuration and add a TLS configuration file:

```bash
mkdir -p dynamic
cat > dynamic/tls.yml << EOF
tls:
  certificates:
    - certFile: /certs/tls.crt
      keyFile: /certs/tls.key
EOF
```

### Create Certificate

#### **Option 1**: Create a Self-Signed Certificate

Generate a self-signed certificate:

```bash
mkdir -p certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/tls.key -out certs/tls.crt \
  -subj "/CN=*.testbed.moh"
```

#### **Option 2**: Create Certificate Using cert-manager

Apply the following YAML to create a certificate in Kubernetes using cert-manager:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: traefik-certificate
  namespace: default
spec:
  secretName: traefik-tls
  issuerRef:
    name: cluster-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
  commonName: "*.testbed.moh"
  dnsNames:
    - "*.testbed.moh"
  usages:
    - digital signature
    - key encipherment
    - server auth
    - client auth
  privateKey:
    algorithm: RSA
    size: 2048
  encodeUsagesInRequest: true
  duration: 8640h # 1 year
  renewBefore: 360h # 15 days
```

Extract the certificate and key from the Kubernetes secret and move them to the `./certs/` directory:

```bash
kubectl get secrets traefik-tls -o jsonpath='{.data.tls\.crt}' | base64 -d > tls.crt
kubectl get secrets traefik-tls -o jsonpath='{.data.tls\.key}' | base64 -d > tls.key
```

### Create Basic Authentication Credentials

Generate a username and password hash using `htpasswd` and add it to the `.env` file:

```bash
htpasswd -nb username "P@ssw0rd" | sed -e 's/\$/\$\$/g'
```

## Installation

Start the Traefik services using Docker Compose:

```bash
docker compose up -d
```

## Resources

- [Traefik Docker Setup](https://doc.traefik.io/traefik/setup/docker/)

- [Exposing Services with Traefik on Docker](https://doc.traefik.io/traefik/expose/docker/)
