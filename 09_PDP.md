# PDP Policy Decision Point

PDP stands for Policy Decision Point and enables controlled access to Service Providers via EngineBlock, before completing the authentication request. Based on free-form rules like attribute values provided by an Identity Provider about a user.
PDP (admin) needs metadata from Service Registry, but the available API does not suffice, so a new API service was created: Metadata Exporter.

We first need to setup Metadata exporter before we can setup PDP.

## Metadata Exporter
Clone build and configure metadata exporter
```
# cd /opt/OpenConext
# git clone https://github.com/OpenConext/OpenConext-metadata-exporter.git
# cd OpenConext-metadata-exporter
# mvn package -D skipTests
...
# cp src/main/resources/application.properties .
# vi application.properties
```
```
server.port=8083
...
spring.datasource.url=jdbc:mysql://localhost/janus_db?autoReconnect=true&useSSL=false
spring.datasource.username=janus
spring.datasource.password=janus_password
...
client.username=metadata.client
client.password=secret
```
Create a link and systemd service start configuration
```
# ln -s target/metadata-exporter-2.0.4-SNAPSHOT.jar metadata-exporter.jar
# vi /etc/systemd/system/metadata-exporter.service
```
```
[Unit]
Description=Metadata Exporter

[Service]
Type=simple
User=root
WorkingDirectory=/opt/OpenConext/OpenConext-metadata-exporter
ExecStart=-/usr/bin/java -Xms64m  -Xmx64m -jar /opt/OpenConext/OpenConext-metadata-exporter/metadata-exporter.jar
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
Enable and start metadata exporter service
```
# systemctl enable metadata-exporter.service
# systemctl start metadata-exporter.service
# tail -f /var/log/syslog
# systemctl status metadata-exporter.service
```
Check proper functioning of Metadata Exporter
```
# curl -su metadata.client:secret -H "Content-Type: application/json" http://localhost:8083/identity-providers.json | jq .
[
  {
    "entityid": "http://engine.myconext.org/authentication/idp/metadata",
    "state": "prodaccepted",
    "allowedall": "yes",
    "attributes": [],
    "certData": "MIIDYDCCAkigAwIBAgIJAP6WN2gy85NtMA0GCSqGSIb3DQEBCwUAMFoxCzAJBgNVBAYTAk5MMRQwEgYDVQQKDAtlbmdpbmVibG9jazEUMBIGA1UEAwwLZW5naW5lYmxvY2sxHzAdBgkqhkiG9w0BCQEWEGluZm9AZW5naW5lYmxvY2swHhcNMTYwNDI2MTMyNjA4WhcNMjYwNDI0MTMyNjA4WjBaMQswCQYDVQQGEwJOTDEUMBIGA1UECgwLZW5naW5lYmxvY2sxFDASBgNVBAMMC2VuZ2luZWJsb2NrMR8wHQYJKoZIhvcNAQkBFhBpbmZvQGVuZ2luZWJsb2NrMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxhdZmuwFkWrd5U9aBOKx0t5rr3lceTW6q7Bx5z39qU1xxwxGpVICP3a4b3lJH9XVRRazd9Vl8d9SXhdJ",
    "SingleSignOnService:0:Binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect",
    "SingleSignOnService:0:Location": "http://engine.myconext.org/authentication/idp/single-sign-on",
    "allowedEntities": []
  },
  {
...
```
## PDP
Now we can install the actual PDP server, which consists of an API server and admin GUI. This installation will, for now, shortcut the SAML authentication in order to have a functional service up and running. It shouldn't be too difficult to extend the apache configuration to SAML authenticated behaviour.

First install required packages
```
# apt-get install npm nodejs-legacy ruby ruby-dev
```
Clone PDP repository and build ruby environment
```
# cd /opt/OpenConext
# git clone https://github.com/OpenConext/OpenConext-pdp.git
# cd OpenConext-pdp
# gem install bundler
# bundle install
```
Create PDP databaes
```
# mysql -p
> CREATE DATABASE pdpserver DEFAULT CHARACTER SET latin1;
> create user 'pdp'@'localhost' identified by 'pdp_password';
> grant all on pdpserver.* to 'pdp'@'localhost';
> flush privileges;
```
PDP build can now be started
```
# mvn package -D skipTests
```
When the build succeeded, PDP-service can be configured
```
# cd pdp-server
# ln -s target/pdp-server-1.1.6.jar pdp-server.jar
# cp src/main/resources/application.properties .
# vi application.properties
```
```
server.port=8084
...
spring.profiles.active=prod
...
spring.datasource.username=pdp
spring.datasource.password=pdp_password
...
xacml.properties.path=file:/opt/OpenConext/OpenConext-pdp/pdp-server/target/classes/xacml.conext.properties
policy.base.dir=file:/opt/OpenConext/OpenConext-pdp/pdp-server/target/classes/xacml/policies
...
metadata.idpRemotePath=http://localhost:8083/identity-providers.json
metadata.spRemotePath=http://localhost:8083/service-providers.json
...
voot.serviceUrl=http://localhost:9090/
voot.accessTokenUri=http://localhost:8081/oauth/token
voot.userAuthorizationUri=http://server.authz.myconext.org/oauth/authorize
voot.clientId=pdp
voot.clientSecret=[Maak een secret in authz-admin voor deze client]
voot.scopes = groups
```
```
# vi target/classes/xacml.conext.properties
```
comment out PIP Engine Definitions
```
...
#
# PIP Engine Definition
#
#xacml.pip.engines=teams,sab
#teams.classname=pdp.teams.TeamsPIP
#sab.classname=pdp.sab.SabPIP
```
Create a PDP server service configuration
```
# vi /etc/systemd/system/pdp-server.service
```
```
[Unit]
Description=PDP Server

[Service]
Type=simple
User=root
WorkingDirectory=/opt/OpenConext/OpenConext-pdp/pdp-server
ExecStart=-/usr/bin/java -Xms64m  -Xmx64m -jar /opt/OpenConext/OpenConext-pdp/pdp-server/pdp-server.jar
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
Enable and start the service
```
# systemctl enable pdp-server.service
# systemctl start pdp-server.service
# tail -f /var/log/syslog
# systemctl status pdp-server.service
```
Add a production IdP with entityID "http://mock-idp" to Service Registry and don't forget to push configuration.

Configure PDP admin GUI
```
# cd /opt/OpenConext/OpenConext-pdp/pdp-gui
# vi Gruntfile.js
```
```
...
    proxies: [
          {
            context: ['/pdp'],
            host: 'localhost',
            port: 8084,
            https: false,
            changeOrigin: false,
            xforward: false
          }
        ]
...
```
And create a systemd service configuration
```
# vi /etc/systemd/system/pdp-gui.service
```
```
[Unit]
Description=PDP GUI

[Service]
Type=simple
User=root
WorkingDirectory=/opt/OpenConext/OpenConext-pdp/pdp-gui
ExecStart=-/opt/OpenConext/OpenConext-pdp/pdp-gui/node_modules/grunt-cli/bin/grunt server
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
Enable and start PGP GUI service
```
# systemctl enable pdp-gui.service
# systemctl start pdp-gui.service
# tail -f /var/log/syslog
# systemctl status pdp-gui.service
```
Create reverse proxy in Apache. Normally you are required to present a valid SAML authentication to use PDP Admin, but to show the functionality I created a fake login by setting the correct Shibboleth headers in the apache vhost conf.
Check the fake Shibboleth data against valid data in Service Registry (Shib-Authenticating-Authority) en connected IdP (name-id).
```
# vi /etc/apache2/sites-available/pdp.conf
```
```
<VirtualHost *:80>
        ServerName pdp.myconext.org
        DocumentRoot /var/www/html

        # Fake Shibboleth SAML login
        RequestHeader set "name-id" "urn:collab:person:myconext.org:student"
        RequestHeader set "email" "s.tudent@myconext.org"
        RequestHeader set "uid" "S.tudent"
        RequestHeader set "displayname" "Stef Student"
        RequestHeader set "schachomeorganization" "myconext.org"
        RequestHeader set "Shib-Authenticating-Authority" "http://idp.myconext.org/simplesaml/saml2/idp/metadata.php"

        ProxyPass / http://localhost:8001/ retry=0
        ProxyPassReverse / http://localhost:8001/
</VirtualHost>
```
Make sure to enable SAML authentication if the admin interface should be used by more than one administrator and access to the interface is unrestricted.

Enable de vhost en reload apache
```
# a2ensite pdp
# service apache2 reload
```
Browse to http://pdp.myconext.org/ (make sure pdp.myconext.org can be resolved to the VM)

### Check PDP functionality

Add a setting "coin:policy_enforcement_decision_required" to the "# Allowed metadata names for SPs" in Service Registry config_janus_core.yml (mind the indentation, it's yml).
```
      # Allowed metadata names for SPs. 
      saml20-sp: 
           # Endpoint fields
           ...

           # COIN control fields 
           coin:policy_enforcement_decision_required: 
               type: boolean 
               default: false 
               default_allow: true 
               required: false
```
Adjust PDP settings for Engineblock in engineblock.ini
```
pdp.baseUrl = "http://localhost:8084/pdp/api/decide/policy"
pdp.username = "pdp-admin"
pdp.password = "secret"
```
Edit a test sp.myconext.org in Service Registry and enable the "coin:policy_enforcement_decision_required" setting (by marking the checkbox). Push configuration.

Create a test PDP Rule in http://pdp.myconext.org/ for this SP and check access using a test user.
Activation of the PAP rule may take some time (refresh interval, default 10m).
