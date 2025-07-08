## Steps:
- Create a private Certificate Authority (CA)
- Generate a server certificate valid for an IP address
- Generate a server certificate valid for an IP address
- Nginx configuration
- `aiohttp` client request with mTLS
### ✳ STEP 1: Create a private Certificate Authority (CA)
```shell
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -sha256 -days 3650 -out ca.crt
```

### ✳ STEP 2: Generate Server Certificate (for domain like deepen.uz)
**Create `openssl.cnf` file:**

```text
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext

[ dn ]
C  = UZ
ST = Tashkent
L  = Tashkent
O  = MyCompany
OU = Dev
CN = server.local

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = example.com
DNS.2 = sub.example.com
IP.1  = 75.22.11.99
```
---
**Generate key and CSR:**
```shell
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
```
---
**Sign the server certificate:**
```shell
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out server.crt -days 3650 -sha256 -extensions req_ext -extfile openssl.cnf
```
### ✳ STEP 3: Generate Client Certificate
```shell
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out client.crt -days 3650 -sha256
```
### ✳ Nginx configuration
```
server {
 listen 443 ssl;
 server_name 192.168.5.236;
 ssl_certificate /etc/nginx/ssl/server.crt;
 ssl_certificate_key /etc/nginx/ssl/server.key;
 ssl_client_certificate /etc/nginx/ssl/ca.crt;
 ssl_verify_client on;
 location / {
 return 200 "Client verified by cert!";
 }
}
```
### ✳ `aiohttp` client request with mTLS
```python
import ssl
import aiohttp
import asyncio


ssl_ctx = ssl.create_default_context(cafile='ca.crt')
ssl_ctx.load_cert_chain(certfile='client.crt', keyfile='client.key')


async def main():
    async with aiohttp.ClientSession() as session:
    async with session.get('https://192.168.5.236:8443', ssl=ssl_ctx) as resp:
    print(await resp.text())
    
asyncio.run(main())
```
---
### ✳ Convert .crt to .pem
```shell
openssl x509 -in server.crt -out server.pem -outform PEM
```
---
### ✳ Check SAN
```shell
openssl x509 -in server.crt -text -noout | grep -A 1 "Subject Alternative Name"
```
