# File Encryption and Decryption

## Encryption

```bash
gpg --symmetric \
    --cipher-algo AES256 \
    --batch --yes \
    --passphrase "P@ssw0rd" \
    before
```

## Decryption

```bash
gpg --output after \
    --decrypt \
    --batch --yes \
    --passphrase "P@ssw0rd" \
    before.gpg
```
