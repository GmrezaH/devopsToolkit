create cert using cert-manager:

```bash
cat <<EOF | kubectl apply -f -
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
EOF
```

save certificate to file and move it to `roles/traefik/files/certs/tls.crt` and `roles/traefik/files/certs/tls.key`:

```bash
kubectl get secrets traefik-tls -o jsonpath='{.data.tls\.crt}' | base64 -d > tls.crt
kubectl get secrets traefik-tls -o jsonpath='{.data.tls\.key}' | base64 -d > tls.key
```

create user and password using `htpasswd`:

```bash
htpasswd -nb admin admin
```

create credentials vault using:

```bash
cat <<EOF > roles/traefik/defaults/main/vault.yml
traefik_web_auth_user: admin
traefik_web_auth_pass: $apr1$MFaT4KkU$fXBqNXCWubXY/RF4r2CLl1
EOF
```

encrypt vault vars:

```bash
ansible-vault encrypt roles/traefik/defaults/main/vault.yml
```

run playbook:

```bash
ansible-playbook playbook/traefik.yml --ask-vault-pass
```
