# OpenSearch And OpenSearch-Dashboard Setup for Production

The following documentation is for basic configuration for production setup.

Install java 17

```
java -version
```
```
apt install openjdk-17-jre-headless
```

Set Java Home

```
readlink -f `which java` | sed "s:/bin/java::"
```

This is a temporary command.
```
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

### For a more permanent JAVA home configuration.
Make backup. 
```
cp ~/.bashrc ~/.bashrc.bak
```

Append using echo.
```
echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> ~/.bashrc
```
### Download OpenSearch
Download both OpenSearch and OpenSearch-Dashboards packages.
```
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.11.1/opensearch-2.11.1-linux-x64.deb
```
```
wget https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.11.1/opensearch-dashboards-2.11.1-linux-x64.deb
```
Extract and install OpenSearch package.
```
sudo dpkg -i opensearch-2.11.1-linux-x64.deb
```
Reload , Enable and Start service.
```
sudo systemctl daemon-reload
```
```
sudo systemctl enable opensearch.service
```
```
sudo systemctl start  opensearch.service
```
Test  connection  and plugins.
```
curl -X GET https://localhost:9200 -u 'admin:admin' --insecure
```
```
curl -X GET https://localhost:9200/_cat/plugins?v -u 'admin:admin' --insecure
```
Edit OpenSearch configuration file.
```
sudo vi /etc/opensearch/opensearch.yml
```
Set Heap.
```
vi /etc/opensearch/jvm.options
```

### Generate a root certificate.

Change directory.
```
cd /etc/opensearch
```
```
sudo rm -f *pem
```
```
sudo openssl genrsa -out root-ca-key.pem 2048
```
```
sudo openssl req -new -x509 -sha256 -key root-ca-key.pem -subj "/C=US/ST=IOWA/L=CEDAR/O=ZITADEL/OU=ADMIN/CN=opensearch.hungry-howard.com" -out root-ca.pem -days 730
```

### Create the admin certificate.

 The admin certs cannot be the same as node certs.
```
sudo openssl genrsa -out admin-key-temp.pem 2048
```
```
sudo openssl pkcs8 -inform PEM -outform PEM -in admin-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out admin-key.pem
```
```
sudo openssl req -new -key admin-key.pem -subj "/C=US/ST=IOWA/L=CEDAR/O=ZITADEL/OU=ADMIN/CN=opensearch.hungry-howard.com" -out admin.csr
```
```
sudo openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out admin.pem -days 730
```

### Create a certificate for the node/s.
```
sudo openssl genrsa -out node1-key-temp.pem 2048
```
```
sudo openssl pkcs8 -inform PEM -outform PEM -in node1-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node1-key.pem
```
```
sudo openssl req -new -key node1-key.pem -subj "/C=US/ST=IOWA/L=CEDAR/O=ZITADEL/OU=ADMIN/CN=opensearch.hungry-howard.com" -out node1.csr
```
```
sudo sh -c 'echo subjectAltName=DNS:opensearch.hungry-howard.com > node1.ext'
```
```
sudo openssl x509 -req -in node1.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node1.pem -days 730 -extfile node1.ext
```
```
sudo chown opensearch:opensearch admin-key.pem admin.pem node1-key.pem node1.pem root-ca-key.pem root-ca.pem
```

## Run the hash script 
This is for Admin password.
```
cd /usr/share/opensearch/plugins/opensearch-security/tools
```
```
./hash.sh
```
Copy the  output and place it in the file.
```
vi /etc/opensearch/opensearch-security/internal_users.yml
```
In  this section.
```
admin:
  	hash: "$2y$12$OKDLCZ1qLXnA5AMLKeKdIueW0e3m1y0nVf/GEBzom1BxNOykpw0Ee"
 	 reserved: true
 	 backend_roles:
 	 - "admin"
  description: "Demo admin user"
```

###  Execute security script.
  
```
./securityadmin.sh -h opensearch.hungry-howard.com  -cd /etc/opensearch/opensearch-security/ -cacert /etc/opensearch/root-ca.pem -cert /etc/opensearch/admin.pem -key /etc/opensearch/admin-key.pem -icl -nhnv
```

Check  the connection/status.
```
curl https://opensearch.hungry-howard.com:9200 -u admin:Password123 -k
```
## OpenSearch-Dashboards

Configure the following in Opensearch-Dashboards.yaml file. 
```
grep -Ev ^'(#|$)' opensearch_dashboards.yml 
```

Example:

This has  the default configurations need for Production setup and also  the configurations needed for SSO  using SAML.

```
---
server.port: 5601
server.host: "opensearch.hungry-howard.com"
server.name: "opensearch.hungry-howard.com"
opensearchDashboards.index: ".opensearch_dashboards"
opensearchDashboards.defaultAppId: "home"
logging.dest: /var/log/opensearch-dashboards/opensearch-dashboards.log
ml_commons_dashboards.enabled: true
opensearch_security.ui.saml.login.buttonname: Zitadel
opensearch_security.auth.type: ["basicauth","saml"]
opensearch_security.auth.multiple_auth_enabled: true
opensearch.hosts: [https://172.31.25.73:9200]
opensearch.ssl.verificationMode: none
opensearch.username: admin
opensearch.password: Password123
opensearch.requestHeadersWhitelist: [authorization, securitytenant]
server.ssl.enabled: true
server.xsrf.allowlist: ["/_opendistro/_security/saml/acs", "/_opendistro/_security/saml/logout"]
opensearch_security.multitenancy.enabled: true
opensearch_security.multitenancy.tenants.preferred: [Private, Global]
opensearch_security.readonly_mode.roles: [kibana_read_only]
opensearch_security.cookie.secure: true
server.ssl.certificate: /etc/opensearch-dashboards/node1.pem
server.ssl.key: /etc/opensearch-dashboards/node1-key.pem
opensearch.ssl.certificateAuthorities: /etc/opensearch-dashboards/root-ca.pem ```
