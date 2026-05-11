# signserver-docker
# Create the CA certificate
mkdir -p certs
openssl genrsa -out certs/ManagementCA.key.pem 2048
openssl req -new -x509 -days 3650 -key certs/ManagementCA.key.pem -out certs/ManagementCA.pem
# Run Docker
sudo docker compose up -d
## export certificat
# Client key
openssl genrsa -out certs/client.key.pem 2048

# Client CSR  
openssl req -new -key certs/client.key.pem -out certs/client.csr.pem \
  -subj "/C=TN/ST=Tunis/L=Tunis/O=MYCERT/OU=PKI/CN=MYCERT"

# Sign with CA
openssl x509 -req -in certs/client.csr.pem -CA certs/ManagementCA.pem \
  -CAkey certs/ManagementCA.key.pem -CAcreateserial -out certs/client.crt.pem -days 365

# Export P12 for browser
openssl pkcs12 -export -in certs/client.crt.pem -inkey certs/client.key.pem \
  -out certs/signserver-admin.p12 -name "MYCERT" -passout pass:mycert

  ## add certificat to browser
  ## ACCESS TO web
  https://localhost:8443/signserver
