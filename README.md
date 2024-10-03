# tyk-mtls-testing

This project does nothing but test the Tyk's mTLS feature on your local machine.

## Required

1. Docker-Compose - Running Tyk
1. OpenSSL - Generate the Public/Private keys
1. Edit your host files to make sure it points back to your own machine. e.g. `127.0.0.1 tyk.local`

## About Certificates

### Server's Certificate (Create a self-signed certificate)

You can skip this part if your server already running HTTPS with publicly trusted certificate.

```
openssl req -x509 -newkey rsa:2048 -keyout ./tyk/certs/server-key.pem -out ./tyk/certs/server-cert.pem -days 365 -nodes -subj "/C=TH/ST=Bangkok/L=Bangkok/O=Muze/OU=Tech Team/CN=tyk.local"
```

### mTLS certificates for testing

### To Create Client's CA Key Pair

```bash
# Create private key using Ellitic Curve algo (to matched Netflix's standard)
openssl ecparam -name prime256v1 -genkey -noout -out ./tyk/certs/rootCA.key

# Generate the Certificate based on given key using Config (rootCA.cnf)
openssl req -x509 -new -key ./tyk/certs/rootCA.key -sha256 -days 3650 -out ./tyk/certs/rootCA.pem

# Verify
openssl x509 -in ./tyk/certs/rootCA.pem -text -noout
```

### To Derive a Client Key Pair

```bash
# Cretae a new (Client) Key
openssl ecparam -name prime256v1 -genkey -noout -out ./tyk/certs/client.key

# Create a CSR
openssl req -new -key ./tyk/certs/client.key -out ./tyk/certs/client.csr -subj "/C=TH/ST=Bangkok/L=Bangkok/O=Client Organization/OU=IT Department/CN=Client1"

# Use CSR to sign against rootCA.pem, and rootCA.key (SHA384)
openssl x509 -req -in ./tyk/certs/client.csr -CA ./tyk/certs/rootCA.pem -CAkey ./tyk/certs/rootCA.key -CAcreateserial -out ./tyk/certs/client.crt -days 365

# Verify
openssl x509 -in ./tyk/certs/client.crt -text -noout
```

## Configure on Tyk

Added to (trusted) client_certificates, this part is actually configured in our docker-compose.

```json
{
    // ...API definitions...,
    "use_keyless": true,
    "use_mutual_tls_auth": true,
    "client_certificates": [
        "/opt/tyk-gateway/certs/rootCA.pem"
    ]
}
```

## Test it out

Make sure your tyk is running using HTTPS, before test it with mTLS.

```bash
docker-compose up tyk -d
```

```bash
curl --cert ./tyk/certs/client.crt --key ./tyk/certs/client.key --cacert ./tyk/certs/server-cert.pem https://tyk.local/bin/get
```
