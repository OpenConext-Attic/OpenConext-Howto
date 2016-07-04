# Authz oAuth server and companion tools
Integration with OpenConext is in the form of SAML authentication on the oAuth consent and admin interface.

Authz service consist of three parts
* OpenConext-authorization-server
  * The oAuth server
* OpenConext-authorization-admin
  * Admin interface to manage oAuth clients
* OpenConext-authorization-playground
  * oAuth client to test oAuth servers

```
# cd /opt/OpenConext
```
## oAuth Server
Authz is, just like VOOT a Java 8 application and is built using maven.
```
# mysql
> CREATE DATABASE `authzserver` DEFAULT CHARACTER SET latin1;
> CREATE USER 'authz'@'localhost' IDENTIFIED BY 'authz_password';
> GRANT ALL ON `authzserver`.* TO 'authz'@'localhost';
> flush privileges;
```
Clone authorization-server
```
# git clone https://github.com/mrvanes/OpenConext-authorization-server.git
# cd OpenConext-authorization-server
# vi ./src/main/resources/application.properties
```
```
server.port=8081
...
spring.datasource.username=authz
spring.datasource.password=authz_password
...
```
Build, link and configure authz-server
```
# mvn package -D skipTests
# ln -s target/authz-server-1.3.4.jar authz-server.jar
# cp ./src/main/resources/application.properties .
```
Create systemd authz-server service
```
# vi /etc/systemd/system/authz-server.service
```
```
[Unit]
Description=Authz Server

[Service]
Type=simple
User=root
ExecStart=-/usr/bin/java -Xms64m  -Xmx64m -jar /opt/OpenConext/OpenConext-authorization-server/authz-server.jar
ExecStop=/bin/kill -TERM $MAINPID

[Install]
WantedBy=multi-user.target
```
Activate and start Authz Service
```
# systemctl enable authz-server.service
# systemctl start authz-server.service
# tail -f /var/log/syslog
# systemctl status authz-server.service
# journalctl -u authz-server.service
```
For SAML authentication a full-fledged Shibboleth installation is preferred. However, for the sake of simplicity a much easier mod_mellon installation with some tweaks is used here.

Make sure you understand the implications of these simplifications, by inspecting the apache configuration files.

Install mod-mellon
```
# apt-get install libapache2-mod-auth-mellon
# a2enmod auth_mellon proxy_http
# a2enmod headers
```
Mellon setup Basis: https://github.com/UNINETT/mod_auth_mellon/wiki/GenericSetup
```
# mkdir -p /etc/apache2/mellon
# cd /etc/apache2/mellon
# wget https://raw.githubusercontent.com/UNINETT/mod_auth_mellon/master/mellon_create_metadata.sh
# chmod 755 mellon_create_metadata.sh
# ./mellon_create_metadata.sh server.authz.myxonext.org http://server.authz.myconext.org/mellon
# chown www-data. server.authz.myxonext.org.key
```
Retrieve Engineblock IdP metadata (engine.myconext.org should resolve from the VM).
```
# wget http://engine.myconext.org/authentication/idp/metadata -O engine_idp.xml
```
Add authz.myconext.org.xml metadata as SP to Service Registry using NameIDFormat=urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified and don't forget to "Push" configuration.
```
# vi /etc/apache2/sites-available/authz.conf
```
```
<VirtualHost *:80>
        ServerName server.authz.myconext.org
        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPass /mellon !
        ProxyPass /mellon.php !
        ProxyPass / http://127.0.0.1:8081/
        ProxyPassReverse / http://127.0.0.1:8081/

        RequestHeader set "name-id" %{MELLON_NAME_ID}e
        RequestHeader set "email" %{MELLON_mail}e
        RequestHeader set "schachomeorganization" %{MELLON_urn:mace:terena.org:attribute-def:schacHomeOrganization}e
        RequestHeader set "Shib-Authenticating-Authority" %{MELLON_IDP}e

        # Allow non-saml access for machine clients.
        <Location ~ "/oauth/(check_token|token)">
                allow from all
                satisfy any
        </Location>

        # Enable shibboleth for all other URLs, but the health check
        <Location ~ "/health">
                allow from all
                satisfy any
        </Location>

        # Enable mellon for all other URLs
        <Location />
                # Add information from the mod_auth_mellon session to the request.
                MellonEnable "auth"
 
                # Configure the SP metadata
                # This should be the files which were created when creating SP metadata.
                MellonSPPrivateKeyFile /etc/apache2/mellon/server.authz.myconext.org.key
                MellonSPCertFile /etc/apache2/mellon/server.authz.myconext.org.cert
                MellonSPMetadataFile /etc/apache2/mellon/server.authz.myconext.org.xml
 
                # IdP metadata. This should be the metadata file you got from the IdP.
                MellonIdPMetadataFile /etc/apache2/mellon/engine_idp.xml
 
                # The location all endpoints should be located under.
                # It is the URL to this location that is used as the second parameter to the metadata generation script.
                # This path is relative to the root of the web server.
                MellonEndpointPath /mellon

                MellonIdP "IDP"
        </Location>
</VirtualHost>
```
Create a simple Mellon test script to inspect apache server variables.
```
# vi /var/www/html/mellon.php
```
```
<?php
phpinfo();

?>
```
Enable authz vhost and restart apache
```
# a2ensite authz
# service apache2 restart
```
Test http://server.authz.myconext.org/mellon.php en check Apache Environment variables starting with MELLON_

authz server also created new database tables
```
# mysql -e "show tables" authzserver
```
To test Authz server we need to add oAuth clients first

## oAuth admin
oAuth admin is the admin interface to the oAuth server configuration database protected by SAML authentication (via token on oAuth-server) and administrative groupmembership by VOOT. This is where the parts start to come together.

The oAuth admin build process awkwardly depends on firefox, which in turn needs a virtual framebuffer (X environment) to start (we're on a headless server).
The Xvfb implementation on Debian does not expose itself over TCP by default, so we need to create a stub that enables a tcp socket.
```
# apt-get install xvfb
# mv /usr/bin/Xvfb /usr/bin/Xvfb.real
# vi /usr/bin/Xvfb
```
```
#!/bin/sh
/usr/bin/Xvfb.real -listen tcp $*
```
```
# chmod 755 /usr/bin/Xvfb

# apt-get install firefox

# cd /opt/OpenConext
# git clone https://github.com/mrvanes/OpenConext-authorization-admin.git
# cd OpenConext-authorization-admin
# vi ./src/main/resources/application.properties
```
```
defaultClientsAndResourceServers.config.path = file:/opt/OpenConext/OpenConext-authorization-admin/clientsAndResources.conf
server.port=8082
allowed_group=urn:collab:group:myconext.org:etc:admin
spring.datasource.username=authz
spring.datasource.password=authz_password
voot.serviceUrl=http://localhost:9090/
voot.redirectUri=http://voot.myconext.org/
voot.accessTokenUri=http://localhost:8081/oauth/token
voot.userAuthorizationUri=http://server.authz.myconext.org/oauth/authorize
```
```
# vi src/main/resources/clientsAndResources.conf
```
```
...
clients = [
  {
    clientId = "authz-playground"
    secret = "secret"
    redirectUri= "http://play.authz.myconext.org/redirect"
    resourceIds = [
      "groups"
      "foo"
    ]
    scopes = [
      "read"
      "write"
      "groups"
      "members"
    ]
    grantTypes= [
      "authorization_code"
      "implicit"
      "client_credentials"
    ]
  }
  {
    clientId = "authz-admin"
    secret = "secret"
    redirectUri= "http://admin.authz.myconext.org/"                                                                                                                                        
    resourceIds = [                                                                                                                                                                  
      "groups"                                                                                                                                                                       
    ]                                                                                                                                                                                
    scopes = [                                                                                                                                                                       
      "groups"                                                                                                                                                                       
    ]                                                                                                                                                                                
    grantTypes = [                                                                                                                                                                   
      "authorization_code"                                                                                                                                                           
      "implicit"
    ]
    autoApprove = "True"
  }
...
```
Build, link and install the server
```
# mvn package -D skipTests
# ln -s target/authz-admin-1.3.6.jar authz-admin.jar
# cp ./src/main/resources/application.properties .
# cp ./src/main/resources/clientsAndResources.conf .
# vi /etc/systemd/system/authz-admin.service
```
```
[Unit]
Description=Authz Admin

[Service]
Type=simple
User=root
ExecStart=-/usr/bin/java -Xms128m  -Xmx128m -jar /opt/OpenConext/OpenConext-authorization-admin/authz-admin.jar
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
Enable and start authz-admin service
```
# systemctl enable authz-admin.service
# systemctl start authz-admin.service
# tail -f /var/log/syslog
# systemctl status authz-admin.service
```
Create Apache vhost (combine with existing authz.conf)
```
# cd /etc/apache2/mellon
# ./mellon_create_metadata.sh admin.authz.myconext.org http://admin.authz.myconext.org/mellon
# vi /etc/apache2/sites-available/authz.conf
```
```
...
</VirtualHost>

<VirtualHost *:80>
        ServerName admin.authz.myconext.org
        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPass /mellon !
        ProxyPass /mellon.php !
        ProxyPass / http://127.0.0.1:8082/
        ProxyPassReverse / http://127.0.0.1:8082/

        RequestHeader set "name-id" %{MELLON_NAME_ID}e
        RequestHeader set "email" %{MELLON_mail}e
        RequestHeader set "uid" %{MELLON_urn:mace:dir:attribute-def:uid}e
        RequestHeader set "displayname" %{MELLON_urn:mace:dir:attribute-def:uid}e
        RequestHeader set "schachomeorganization" %{MELLON_urn:mace:terena.org:attribute-def:schacHomeOrganization}e
        RequestHeader set "Shib-Authenticating-Authority" %{MELLON_IDP}e

        # Enable shibboleth for all other URLs, but the health check
        <Location ~ "/health">
                allow from all
                satisfy any
        </Location>

        # Enable mellon for all other URLs
        <Location />
                # Add information from the mod_auth_mellon session to the request.
                MellonEnable "auth"

                # Configure the SP metadata
                # This should be the files which were created when creating SP metadata.
                MellonSPPrivateKeyFile /etc/apache2/mellon/admin.authz.myconext.org.key
                MellonSPCertFile /etc/apache2/mellon/admin.authz.myconext.org.cert
                MellonSPMetadataFile /etc/apache2/mellon/admin.authz.myconext.org.xml

                # IdP metadata. This should be the metadata file you got from the IdP.
                MellonIdPMetadataFile /etc/apache2/mellon/engine_idp.xml

                # The location all endpoints should be located under.
                # It is the URL to this location that is used as the second parameter to the metadata generation script.
                # This path is relative to the root of the web server.
                MellonEndpointPath /mellon

                MellonIdP "IDP"
        </Location>
</VirtualHost>
```
Based on the clientsAndResources.conf Authz-admin will automatically create new clients in the authzserver database
```
# mysql
> select client_id,resource_ids,web_server_redirect_uri  from authzserver.oauth_client_details where client_id='authz-admin';
+-------------+--------------+------------------------------+
| client_id   | resource_ids | web_server_redirect_uri      |
+-------------+--------------+------------------------------+
| authz-admin | groups       | http://admin.openconext-auth |
+-------------+--------------+------------------------------+
1 row in set (0.00 sec)
```
Add admin.authz.myconext.org metadata xml in /etc/apache2/mellon to Service Registry using NameIDFormat=urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified and don't forget to "Push" configuration.

In order to have a valid admin user, create an admin group under the 'etc' folder in Grouper and make some administrator account that has authenticated via Engineblock at least once member of this group.
Adjust application.properties allowed_group properties to reflect membership of this group. Remember that VOOT will prefix the retrieved group with urn:collab.group:myconext: and Grouper will prefix group id with etc:

Auth-server-admin is the most complext application in the OpenConext landscape until now, because it consumes all previously described parts to make it work.
It is protected by SAML connection with Engineblock, consults VOOT for proper groupmembership, VOOT in it's turn consults Authz-server to request for permission to get groupmemberhip information on behalf of the authenticated user.

## oAuth playground
oAuth playground is used to test requesting and using oAuth tokens form oAuth server
```
# cd /opt/OpenConext
# git clone https://github.com/mrvanes/OpenConext-authorization-playground.git
# cd OpenConext-authorization-playground
# vi src/main/resources/application.properties
```
```
oauth.redirect_uri = http://play.authz.myconext.org/redirect
oauth.token_uri = http://localhost:8081/oauth/token
oauth.client_id = authz-playground
oauth.client_secret = secret
oauth.scopes = read write groups members
oauth.authorize_url = http://server.authz.myconext.org/oauth/authorize
oauth.resource_server_api_url = http://localhost:9090/me/groups
oauth.check_token_url = http://localhost:8081/oauth/check_token
```
```
# mvn package -D skipTests
# ln -s target/authz-playground-1.2.jar authz-playground.jar
# cp src/main/resources/application.properties .

# vi /etc/systemd/system/authz-play.service
```
```
[Unit]
Description=Authz Playground

[Service]
Type=simple
User=root
WorkingDirectory=/opt/OpenConext/OpenConext-authorization-playground
ExecStart=-/usr/bin/java -Xms64m  -Xmx64m -jar /opt/OpenConext/OpenConext-authorization-playground/authz-playground.jar
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
Enable and start service
```
# systemctl enable authz-play.service 
# systemctl start authz-play.service
# tail -f /var/log/syslog
# systemctl status authz-play.service

# vi /etc/apache2/sites-available/authz.conf
```
```
...
</VirtualHost>
<VirtualHost *:80>
        ServerName authp.myconext.org
        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPass / http://127.0.0.1:8089/
        ProxyPassReverse / http://127.0.0.1:8089/
</VirtualHost>
```
```
# service apache2 reload
```
Test http://play.authz.myconext.org/ (make sure play.authz.myconext.org resolves to your VM or server)

Most fields will be pre-filled
```
OAuth Client Id
    play-client
OAuth secret
    secret
OAuth scopes (space-separated, case-sensitive)
    read write groups members
OAuth grant type
    Authorization code
Use no redirect URI
    false (uit)
AccessToken URL
    http://localhost:8081/oauth/token
Authorization URL
    http://server.authz.myconext.org/oauth/authorize
```
[Next]
```
Authorization URL
    http://server.authz.myconext.org/oauth/authorize?response_type=code&client_id=test_client&scope=read+groups+members&state=example&redirect_uri=http://play.openconext-auth/redirect
```
[Next]
```
HTTP Response Headers
    Status: 200
    ...
HTTP Response Body
    {
    "access_token" : "bla bla bla"
    ...
    "scope" : "groups members read"
    }
```
[Fetch]
```
HTTP Response Body
    [ {
    "id" : "...
    ...
    "membership" : {...
    ...
    } ]
```

Next chapter: [PDP](https://github.com/mrvanes/OpenConext-Howto/blob/master/09_PDP.md)
