# Install OpenSSL in your system to generate selfsigned Server Certificate.

# How to Generate Server certificate and Server Encrypted for JWT login using openssl.
- Step1: openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
- Step2: openssl rsa -passin pass:x -in server.pass.key -out server.key
- Step3: openssl req -new -key server.key -out server.csr
- Step4: openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt

- Step 5:
## Generate the KEY & IV : 
openssl enc -aes-256-cbc -k GITHUBACTIONSMYDEV -P -md sha1 -nosalt

- Step 6 To Encrypt Server.key File 
 openssl enc -nosalt -aes-256-cbc -in server.key -out server.key.enc -base64 -K <KEY> -iv <IV>

## Verify the Login to Salesforce using Consumer Key
sf org login jwt --username vneha@test.com --jwt-key-file assets/dev/server.key --client-id <Consumer Key from Connected App> --set-default --alias ci-org --instance-url https://test.salesforce.com

- Encrypted file location to be strored in environmental variables :  "assets/dev/server.key.enc"