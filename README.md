# Prisma Cloud Private Registry Example

This example shows how you can configure Prisma Cloud to scan private registries.

**Note:** No inbound internet access is required, nor do any ports need to be opened. The scans will run successfully as long as one or more Defenders can reach the registry.

## Set up registry

```
REGISTRY_USERNAME=testuser
REGISTRY_PASSWORD=testpassword
CERTS_DIR=/tmp/reg/certs
AUTH_DIR=/tmp/reg/auth

mkdir -p $CERTS_DIR
mkdir -p $AUTH_DIR

docker run \
  --entrypoint htpasswd \
  httpd:2 -Bbn $REGISTRY_USERNAME $REGISTRY_PASSWORD > $AUTH_DIR/htpasswd

openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout $CERTS_DIR/domain.key -x509 -days 365 \
  -out $CERTS_DIR/domain.crt

docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v $AUTH_DIR:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v $CERTS_DIR:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2

docker login localhost:5000 -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD
docker pull python
docker tag python localhost:5000/python_private_image
docker push localhost:5000/python_private_image
```

## Set up Prisma Cloud

1. Deploy the Prisma Cloud Defender onto the EC2 instance
2. Create a Collection that includes the EC2 instance
3. Navigate to "Compute" -> "Defend" -> "Vulnerabilities" -> "Images" -> "Registry Settings", then click "Add registry"
4. Select "Docker Registry v2" from the "Version" dropdown
5. Enter "localhost:5000" in the "Registry" field
6. Select "Add new credential" from the "Credential" dropdown
7. Enter the registry username & password you specified above
8. Specify your Collection in the "Scanner scope" field
9. Click "Add", then click "Save and scan"
10. Navigate to "Compute" -> "Monitor" -> "Vulnerabilities" -> "Images" -> "Registry", then set the "Registry" filter to `localhost:5000`

This will display your private registry scan results.