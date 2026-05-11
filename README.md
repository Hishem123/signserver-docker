# signserver-docker
#Create the CA certificate
mkdir -p certs
openssl genrsa -out certs/ManagementCA.key.pem 2048
openssl req -new -x509 -days 3650 -key certs/ManagementCA.key.pem -out certs/ManagementCA.pem
sudo docker compose up -d
