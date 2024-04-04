# OpenSearch And OpenSearch-Dashboard Setup for Production

The following documentation is for basic configuration for production setup.

### Download OpenSearch

Change directory

```
cd /home
```
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

Reload, Enable and Start service.

```
sudo systemctl daemon-reload
```
```
sudo systemctl enable opensearch.service
```

Install java 17

By Default JAVA 17 is installed.

Check status

```
java -version
```

Set Java Home.

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

Start Opensearch.

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
sudo openssl req -new -x509 -sha256 -key root-ca-key.pem -subj "/C=US/ST=IOWA/L=CEDAR/O=ZITADEL/OU=ADMIN/CN=ROOT" -out root-ca.pem -days 730
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
sudo openssl req -new -key admin-key.pem -subj "/C=US/ST=IOWA/L=CEDAR/O=ZITADEL/OU=ADMIN/CN=ADMIN" -out admin.csr
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
sudo openssl req -new -key node1-key.pem -subj "/C=US/ST=IOWA/L=CEDAR/O=ZITADEL/OU=ADMIN/CN=opensearch.domain.com" -out node1.csr
```
```
sudo sh -c 'echo subjectAltName=DNS:opensearch.domain.com > node1.ext'
```
```
sudo openssl x509 -req -in node1.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node1.pem -days 730 -extfile node1.ext
```
```
sudo chown opensearch:opensearch admin-key.pem admin.pem node1-key.pem node1.pem root-ca-key.pem root-ca.pem
```
###   Opensearch Configuration file

Edit OpenSearch configuration file.

```
sudo vi /etc/opensearch/opensearch.yml
```

The configuration below also shows the settings needed for SSO using SAML.

```
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch
network.host: 172.31.25.73 
http.port: 9200
discovery.type: single-node
plugins.security.disabled: false
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.http.enabled: true
plugins.security.allow_unsafe_democertificates: true
plugins.security.allow_default_init_securityindex: true
plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".plugins-ml-config", ".plugins-ml-connector", ".plugins-ml-model-group", ".plugins-ml-model", ".plugins-ml-task", ".plugins-ml-conversation-meta", ".plugins-ml-conversation-interactions", ".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opensearch-notifications-*", ".opensearch-notebooks", ".opensearch-observability", ".ql-datasources", ".opendistro-asynchronous-search-response*", ".replication-metadata-store", ".opensearch-knn-models", ".geospatial-ip2geo-data*"]
node.max_local_storage_nodes: 3
plugins.security.ssl.transport.pemcert_filepath: /etc/opensearch/node1.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/opensearch/node1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/opensearch/root-ca.pem
plugins.security.ssl.http.pemcert_filepath: /etc/opensearch/node1.pem
plugins.security.ssl.http.pemkey_filepath: /etc/opensearch/node1-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/opensearch/root-ca.pem
plugins.security.authcz.admin_dn:
  - 'CN=ADMIN,OU=ADMIN,O=ZITADEL,L=CEDAR,ST=IOWA,C=US'
plugins.security.nodes_dn:
  - 'CN=opensearch.domain.com,OU=ADMIN,O=ZITADEL,L=CEDAR,ST=IOWA,C=US'
```
###  Run the hash script 

This is for New User password.

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
  hash: "$2y$12$5oHqYFJAIhAO2w6s3ppROOy8WliwBnL.5uVpKTuGySnmbH3EyNZrW"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"
```
Restart Opensearch service.
```
systemctl restart opensearch
```

###  Execute security script.
  
```
./securityadmin.sh -h opensearch.domain.com  -cd /etc/opensearch/opensearch-security/ -cacert /etc/opensearch/root-ca.pem -cert /etc/opensearch/admin.pem -key /etc/opensearch/admin-key.pem -icl -nhnv
```

Check  the connection/status.
```
curl https://opensearch.domain.com:9200 -u admin:Password123 -k
```
## OpenSearch-Dashboards

Change directory

```
cd /home
```

Install Opensearch-Dashboards

```
sudo dpkg -i opensearch-dashboards-2.11.1-linux-x64.deb
```

Reload systemd

```
sudo systemctl daemon-reload
```
Enable Opensearch-Dashboards. Start Opensearch-Dashboards after the configuration file is set.

```
sudo systemctl enable opensearch-dashboards.service
```


Configure the following in Opensearch-Dashboards.yaml file. 
```
vi /etc/opensearch-dashboards/opensearch_dashboards.yml
```
(Optional) check opensearch-dashboards.yml file.
 
```
grep -Ev ^'(#|$)' /etc/opensearch-dashboards/opensearch_dashboards.yml 
```

Example:

This has the default configurations need for Production setup and also  the configurations needed for SSO  using SAML.

```
---
server.port: 5601
server.host: "opensearch.domain.com"
server.name: "opensearch.domain.com"
### opensearchDashboards.index: ".opensearch_dashboards" <--- this should be  commented out , if not reports  do not work ###########
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
opensearch.ssl.certificateAuthorities: /etc/opensearch-dashboards/root-ca.pem
```

Copy Node and Root Certificates and place them in Opensearch-Dashboards directory.

Change directory.

```
cd /etc/opensearch
```
Copy certificates.

```
cp node1-key.pem node1.pem root-ca.pem /etc/opensearch-dashboards/
```
Change directory. 

```
cd /etc/opensearch-dashboards/
```

Add Permission

```
chown opensearch-dashboards:opensearch-dashboards node1-key.pem node1.pem root-ca.pem
```

Incert certifictae in keystory
```
keytool -import -alias opensearch.domain.com  -file root-ca.pem  -keystore /usr/lib/jvm/java-17-openjdk-amd64/lib/security/cacerts -storepass changeit
```
### Restart Services

```
systemctl restart opensearch
```
Opensearch-dashboards
```
systemctl restart opensearch-dashboards
```
