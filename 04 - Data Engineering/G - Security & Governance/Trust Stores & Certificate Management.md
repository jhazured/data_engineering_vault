# Trust Stores & Certificate Management

**Tags:** #security #ssl #tls #certificates #trust-store #https #encryption

## Overview

A trust store is a repository (database or file) that contains trusted digital certificates — specifically, the public certificates of Certificate Authorities (CAs) or other entities you explicitly trust. Trust stores are fundamental to secure communication in data engineering pipelines that connect to APIs, databases, and cloud services over HTTPS/TLS.

---

## How Trust Stores Work

When a client (your code, Postman, curl, etc.) makes an HTTPS request, a TLS handshake happens:

1. **Client sends request** to `https://api.example.com`
2. **Server sends its certificate** to prove its identity
3. **Client checks the certificate** against its trust store:
   - Is the certificate signed by a trusted CA?
   - Does the certificate chain up to a trusted root in the trust store?
4. **If trusted** — connection is allowed
5. **If not trusted** — client blocks the request with a trust error (e.g. `SSLHandshakeException`)

---

## Where Trust Stores Are Used

| System | Trust Store Location |
|--------|---------------------|
| Web browsers | Built-in (Chrome, Firefox each maintain their own) |
| Java applications | `cacerts` file in `$JAVA_HOME/lib/security/` |
| Operating systems | System-wide certificate store (Windows Certificate Manager, macOS Keychain) |
| Python (requests) | `certifi` package or system CA bundle |
| APIs and clients | Verify server certs before allowing connections |
| Mutual TLS (mTLS) | Trust stores on both sides to verify certificates |

---

## Common Certificate Formats

| Format | Extension | Description |
|--------|-----------|-------------|
| Java KeyStore | `.jks` | Java-specific binary format |
| PKCS12 | `.p12`, `.pfx` | Cross-platform; used by browsers, curl, etc. |
| PEM | `.pem`, `.crt` | Base64-encoded; human-readable; most common on Linux |
| DER | `.der`, `.cer` | Binary-encoded; compact but not human-readable |

---

## Creating Self-Signed Certificates (OpenSSL)

For testing or local HTTPS development:

```bash
# 1. Generate a private key
openssl genrsa -out mysite.key 2048

# 2. Generate a certificate signing request (CSR)
openssl req -new -key mysite.key -out mysite.csr

# 3. Generate the self-signed certificate (valid for 365 days)
openssl x509 -req -days 365 -in mysite.csr -signkey mysite.key -out mysite.crt
```

### Combined PEM File

Some systems (like Nginx) use a combined `.pem` file:

```bash
cat mysite.crt mysite.key > mysite.pem
```

### Using in Python (Flask)

```python
app.run(ssl_context=('mysite.crt', 'mysite.key'))
```

---

## Java Keystore Management (keytool)

`keytool` is a Java command-line tool bundled with every JDK/JRE for managing keystores and trust stores.

### Create a Keystore with Self-Signed Certificate

```bash
keytool -genkeypair \
  -alias mycert \
  -keyalg RSA \
  -keysize 2048 \
  -validity 365 \
  -keystore mykeystore.jks
```

### List Certificates in a Keystore

```bash
keytool -list -v -keystore mykeystore.jks
```

### Export a Certificate

```bash
keytool -exportcert \
  -alias mycert \
  -file mycert.cer \
  -keystore mykeystore.jks \
  -rfc
```

### Import a Trusted Certificate into a Trust Store

```bash
keytool -importcert \
  -alias trusted-cert \
  -file server.crt \
  -keystore truststore.jks
```

If the keystore does not exist, it will be created. You will be prompted to trust the certificate.

### Convert to PKCS12 Format

```bash
keytool -importkeystore \
  -srckeystore mykeystore.jks \
  -destkeystore mycert.p12 \
  -deststoretype PKCS12
```

---

## Mutual TLS (mTLS) Workflow

1. Generate a keystore and client certificate with `keytool`
2. Export the public certificate
3. Import it into the server's trust store
4. Import the server cert into your trust store
5. Use the `.p12` file with an HTTPS client (curl or Postman)

---

## Common Issues & Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `SSLHandshakeException` | Server cert not in trust store | Import the server's CA cert into your trust store |
| `PKIX path building failed` | Incomplete certificate chain | Import intermediate CA certs |
| `certificate not trusted` | Self-signed cert or unknown CA | Add the cert to the trust store manually |
| `certificate has expired` | Cert past its validity date | Renew the certificate |
| Connection refused on port 443 | TLS not configured on server | Check server TLS configuration |

---

## Security Best Practices

- **Never use the default Java trust store password** (`changeit`) in production
- **Rotate certificates** before expiry — set up monitoring for cert expiration
- **Use CA-signed certificates** in production; self-signed only for development
- **Pin certificates** in high-security environments to prevent MITM attacks
- **Store private keys securely** — use AWS Secrets Manager, Azure Key Vault, or HashiCorp Vault
- **Separate keystores from trust stores** — keystores hold your private keys; trust stores hold public CA certs

---

## Data Engineering Context

Trust stores are relevant whenever pipelines connect to:

- **Cloud APIs** (AWS, GCP, Azure) — managed by SDK/certifi, but custom endpoints may need manual trust
- **On-premise databases** behind corporate CAs — require importing internal CA certs
- **SFTP/FTPS servers** — [[GoAnywhere MFT]] and similar tools use Java trust stores
- **SOAP web services** — [[SOAP (Simple Object Access Protocol)|SOAP]] APIs over HTTPS require valid server certificates
- **REST APIs** — [[REST APIs]] over HTTPS; OAuth token endpoints require trusted TLS

---

**Related:** [[REST APIs]] | [[SOAP (Simple Object Access Protocol)|SOAP]] | [[Snowflake RBAC & Data Security]]
