# File Encryption and Decryption in Linux

This guide provides instructions for encrypting and decrypting files using GPG (GNU Privacy Guard) on Linux systems. GPG supports symmetric encryption with AES256 for secure file protection in DevOps workflows, including configuration files or sensitive deployment artifacts.

## Prerequisites

- Install GPG if not already available:

  - **On RHEL/Rocky Linux**:
    ```bash
    sudo dnf install -y gnupg2
    ```
  - **On Debian/Ubuntu**:
    ```bash
    sudo apt update
    sudo apt install -y gnupg
    ```

- Replace `"P@ssw0rd"` in the examples with a strong, secure passphrase.
- Ensure the target file (`unencrypted_file`) exists in the current directory.

## Encryption

Encrypt a file using symmetric encryption with AES256. This creates an encrypted output file with a `.gpg` extension.

```bash
gpg --symmetric \
    --cipher-algo AES256 \
    --batch --yes \
    --passphrase "P@ssw0rd" \
    unencrypted_file
```

- **Output**: Generates `unencrypted_file.gpg`.
- **Notes**: Use `--batch --yes` for non-interactive mode, suitable for scripts or automation.

## Decryption

Decrypt the encrypted file back to its original form.

```bash
gpg --output decrypted_file \
    --decrypt \
    --batch --yes \
    --passphrase "P@ssw0rd" \
    unencrypted_file.gpg
```

- **Output**: Generates `decrypted_file` with the original content.
- **Notes**: Verify the decrypted file matches the original using tools like `diff` or checksums (e.g., `sha256sum`).

## Best Practices

- Store passphrases securely, such as in a password manager or environment variables for CI/CD pipelines.
- For production use, integrate with tools like Ansible Vault for managing encrypted secrets in DevOps environments.
- Regularly rotate passphrases and audit access to encrypted files.
