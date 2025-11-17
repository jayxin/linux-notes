# GPG

## Encrypt a file with just password

```bash
# Encrypt MY_FILE into MY_FILE.gpg:
gpg --batch -c --passphrase MY_PASSPHRASE --output MY_FILE.gpg MY_FILE

# Decrypt MY_FILE.gpg into MY_FILE:
gpg --batch -d --passphrase MY_PASSPHRASE --output MY_FILE MY_FILE.gpg
```
