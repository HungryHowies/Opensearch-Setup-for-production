# Opensearch And Opensearch-dashboard-setup-for-production
Basic  configuration for production setup for SAML /SSO  using Zitadel
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
This is temp,
```
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```
Make backup. 
```
cp ~/.bashrc ~/.bashrc.bak
```

Append using echo.
```
echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> ~/.bashrc
```
```
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.11.1/opensearch-2.11.1-linux-x64.deb
```
```
wget https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.11.1/opensearch-dashboards-2.11.1-linux-x64.deb
```
```
sudo dpkg -i opensearch-2.11.1-linux-x64.deb
```
```
sudo systemctl daemon-reload
```
```
sudo systemctl enable opensearch.service
```
```
sudo systemctl start  opensearch.service
```
```
curl -X GET https://localhost:9200 -u 'admin:admin' --insecure
```
```
curl -X GET https://localhost:9200/_cat/plugins?v -u 'admin:admin' --insecure
```
```
sudo vi /etc/opensearch/opensearch.yml
```
```
vi /etc/opensearch/jvm.options
```
```
cd /etc/opensearch
```
```
sudo systemctl daemon-reload
```
```
sudo systemctl enable opensearch.service
```

Generate a root certificate.
```
sudo rm -f *pem
```
```
sudo openssl genrsa -out root-ca-key.pem 2048
```
```
sudo openssl req -new -x509 -sha256 -key root-ca-key.pem -subj "/C=US/ST=IOWA/L=CEDAR/O=ZITADEL/OU=ADMIN/CN=opensearch.hungry-howard.com" -out root-ca.pem -days 730
```

create the admin certificate.
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

Create a certificate for the node.
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

running hash script fix dup lines.
```
cd /usr/share/opensearch/plugins/opensearch-security/tools
```
```
./hash.sh
```

Execute security script.
```
./securityadmin.sh -h opensearch.hungry-howard.com  -cd /etc/opensearch/opensearch-security/ -cacert /etc/opensearch/root-ca.pem -cert /etc/opensearch/admin.pem -key /etc/opensearch/admin-key.pem -icl -nhnv
```

Status
```
curl https://opensearch.hungry-howard.com:9200 -u admin:Password123 -k
```
## Opensearch-Dashboards
Configure the following.

```
grep -Ev ^'(#|$)' opensearch_dashboards.yml
```
```
---
server.port: 5601
server.host: "opensearch.hungry-howard.com"
server.name: "opensearch.hungry-howard.com"
logging.dest: /var/log/opensearch-dashboards/opensearch-dashboards.log
opensearch.hosts: [https://opensearch.hungry-howard.com:9200]
opensearch.ssl.verificationMode: full
opensearch.username: admin
opensearch.password: Password123
opensearch.requestHeadersWhitelist: [authorization, securitytenant]
server.ssl.enabled: true
opensearch_security.multitenancy.enabled: true
opensearch_security.multitenancy.tenants.preferred: [Private, Global]
opensearch_security.readonly_mode.roles: [kibana_read_only]
opensearch_security.cookie.secure: false
server.ssl.certificate: /etc/opensearch-dashboards/node1.pem
server.ssl.key: /etc/opensearch-dashboards/node1-key.pem
opensearch.ssl.certificateAuthorities: [ "/etc/opensearch-dashboards/root-ca.pem"]
```
