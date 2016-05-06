# Engineblock
Engineblock is the core SAML engine of OpenConext and fullfils the hub functionality of what SAML defines as a hub-and-spoke federation.

At the moment of writing Engineblock still has a hard dependancy to a functional directory server (LDAP). It uses this directory to store previously seen user attributes so that it is able to (re)request consent when any of the attributes changed since the last the user traversed Engineblock.
We will use the default Ubuntu OpenLDAP installation use for the sake of simplicity use the default admin user to connect from Engineblock. This is not a recommended setup for production scenario's and left as an excersise to the reader.

## OpenLDAP
Reading up on Ubuntu OpenLDAP installation how-to's it seems important to have a valid line for the organisation FQDN pointing to 127.0.1.1
```
# vi /etc/hosts
127.0.1.1   myconext.org myconext
```
Now the openldap installation can begin
```
# apt-get install slapd ldap-utils
# dpkg-reconfigure slapd
	DNS: myconext.org
	Organisation name: myconext
```
LDAP is now at least bootstrapped with a tree 'dc=myconext,dc=org' and an admin account 'cn=admin,dc=myconext,dc=org'.
Check the validity of the account using the commandline below, replacing the password with whatever was used during the installation.
```
# ldapsearch -H ldap://localhost -D "cn=admin,dc=myconext,dc=org" -w admin -b "dc=myconext,dc=org"
```
The output should contain the newly created and for the command used cn=admin user.

Install php ldap modules and restart apache
```
# apt-get install php-ldap
# service apache2 restart
```
## Engineblock
Make a clone of an engineblock repository, either a fork or the real thing.
```
# cd /opt/OpenConext
# git clone https://github.com/mrvanes/OpenConext-engineblock.git
# ln -s OpenConext-engineblock engineblock
# cd engineblock
```
Engineblock gets it's configuration out-of-tree from a pre-defined system configuration directory
```
# mkdir -p /etc/openconext
# cp docs/example.engineblock.ini /etc/openconext/engineblock.ini
```
Create a log directory
```
# mkdir -p /var/log/engineblock
# chown www-data. /var/log/engineblock
```
Bootstrap database
```
# mysql
> CREATE DATABASE engineblock DEFAULT CHARSET utf8 DEFAULT COLLATE utf8_unicode_ci;
> CREATE USER 'engineblock'@'localhost' IDENTIFIED BY 'eb_password';
> GRANT ALL PRIVILEGES ON engineblock.* TO 'engineblock'@'localhost' WITH GRANT OPTION;
> flush privileges;
```
For Engineblock to do it's work we need to import some extra directory schemas
```
# cd ldap/schemas
# ldapadd -Y EXTERNAL -H ldapi:/// -f eduperson-200412.ldif
# ldapadd -Y EXTERNAL -H ldapi:/// -f nleduperson.ldif
# ldapadd -Y EXTERNAL -H ldapi:/// -f collab.ldif
# cd ../..
```
There are some php(5) dependancy problems using the default simpleSAMLphp version required by engineblock in Ubuntu Xenial, which can be solved by editing composer.json
```
-        "simplesamlphp/simplesamlphp":      "~1.13",
+        "simplesamlphp/simplesamlphp":      "~1.14",
```
... and remove composer lockfile
```
# rm composer.lock
```
As opposed to Janus, Engineblock comes with it's own composer.phar so we don't need to download
```
# bin/composer.phar install
```
Adjust the engineblock defaults in system directory location
```
# vi /etc/openconext/engineblock.ini
hostname = engine.myconext.org
auth.simplesamlphp.idp.entityId = http://engine.myconext.org/authentication/idp/metadata
auth.simplesamlphp.idp.location = http://engine.myconext.org/authentication/idp/single-sign-on

cookie.lang.domain = engine.myconext.org

database.master1.dsn = "mysql:host=localhost;dbname=engineblock"
database.master1.password = "eb_password"
database.master1.user = "engineblock"
database.masters[] = master1
database.slaves[] = master1

#debug = false
debug = true

email.idpDebugging.to.address = beheerder@myconext.org

ldap.host = myconext.org
ldap.port = 389
ldap.userName = "cn=admin,dc=myconext,dc=org"
ldap.password = "admin"
ldap.baseDn   = "dc=myconext,dc=org"
ldap.useSsl   = false

phpSettings.display_errors = true
phpSettings.date.timezone = "Europe/Amsterdam"

metadataRepositories[0] = database
metadataRepository.database.type = Doctrine

# Janus API settings tbv profile
metadataRepositories[1] = janus
metadataRepository.janus.type = JanusRestV1
serviceRegistry.location    =	"http://serviceregistry.myconext.org/serviceregistry/module.php/janus/services/rest/"
serviceRegistry.user        = "engineblock"
serviceRegistry.user_secret = "eb_password"

engineApi.user = janus
engineApi.password = janus_password
```
Add serviceregistry.myconext.org to the VM's /etc/hosts file
```
127.0.1.1	myconext.org myconext serviceregistry.myconext.org
```
See application/configs/application.ini voor other overridable settings.

Create (self-signed) engineblock SAML TS cert and key in /etc/openconext/
```
# ls -1 /etc/openconext
engineblock.crt
engineblock.csr
engineblock.ini
engineblock.key
```
Create a couple of apache vhost entries for api.myconext.org, engine.myconext.org and profile.myconext.org.
static.myconext.org is optionally used for serving static content like SP and IdP logo's in the WAYF (Where Are You From page).
```
# vi /etc/apache2/sites-available/engineblock.conf
<VirtualHost *:80>
        ServerName api.myconext.org
        DocumentRoot /opt/OpenConext/engineblock/www/api

        RewriteEngine On
        # We support only GET/POST/HEAD
        RewriteCond %{REQUEST_METHOD} !^(POST|GET|HEAD)$
        RewriteRule .* - [R=405,L]
        # If the requested url does not map to a file or directory, then forward it to index.php/URL.
        # Note that it MUST be index.php/URL because Corto uses the PATH_INFO server variable
        RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-f
        RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-d
        RewriteRule ^(.*)$ /index.php/$1 [L] # Send the query string to index.php

        # Requests to the domain (no query string)
        RewriteRule ^$ index.php/ [L]
</VirtualHost>
<VirtualHost *:80>
        ServerName engine.myconext.org
        DocumentRoot /opt/OpenConext/engineblock/www/authentication

        RewriteEngine On
        # We support only GET/POST/HEAD
        RewriteCond %{REQUEST_METHOD} !^(POST|GET|HEAD)$
        RewriteRule .* - [R=405,L]
        # If the requested url does not map to a file or directory, then forward it to index.php/URL.
        # Note that it MUST be index.php/URL because Corto uses the PATH_INFO server variable
        RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-f
        RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-d
        RewriteRule ^(.*)$ /index.php/$1 [L] # Send the query string to index.php

        # Requests to the domain (no query string)
        RewriteRule ^$ index.php/ [L]
</VirtualHost>
<VirtualHost *:80>
        ServerName profile.myconext.org
        DocumentRoot /opt/OpenConext/engineblock/www/profile

        Alias /simplesaml /opt/OpenConext/engineblock/vendor/simplesamlphp/simplesamlphp/www

        RewriteEngine On
        # We support only GET/POST/HEAD
        RewriteCond %{REQUEST_METHOD} !^(POST|GET|HEAD)$
        RewriteRule .* - [R=405,L]
        # If the requested url does not map to a file or directory, then forward it to index.php/URL.
        # Note that it MUST be index.php/URL because Corto uses the PATH_INFO server variable
        RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-f
        RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-d
        RewriteCond %{REQUEST_URI} !^/(simplesaml.*)$
        RewriteRule ^(.*)$ /index.php/$1 [L] # Send the query string to index.php

        # Requests to the domain (no query string)
        RewriteRule ^$ index.php/ [L]
</VirtualHost>
<VirtualHost *:80>
        ServerName static.myconext.org
        DocumentRoot /var/www/static
</VirtualHost>
```
Enable this new apache vhost and reload apache
```
# a2ensite engineblock.conf
# service apache2 reload
```
Disable secure cookies (for now, take care of a secure installtion in production)
```
# vi application/configs/application.ini
[base]
+use_secure_cookies = false
```
Migrate creates all required database tables
```
# cd bin
# ./migrate
```
Bootstrap Engineblock metadata in ServiceRegistry
Engineblock SP:
```
entityID = http://engine.myconext.org/authentication/sp/metadata
type = saml20-sp
AssertionConsumerService:0:Binding = urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST
AssertionConsumerService:0:Location = http://engine.myconext.org/authentication/sp/consume-assertion
NameIDFormat = urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified
certData = [see /etc/openconext/engine.crt, paste cert as one line]
```
Engineblock IdP
```
entityID = http://engine.myconext.org/authentication/idp/metadata
type = saml20-idp
SingleSignOnService:0:Binding = urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect
SingleSignOnService:0:Location = http://engine.myconext.org/authentication/idp/single-sign-on
certData = [see /etc/openconext/engine.crt, paste cert as one line]
```
Enable Engineblock Push synchronisation in ServiceRegistry
```
# vi /opt/OpenConext/serviceregistry/modules/janus/app/config/config_janus_core.yml
    # Allow manual (or future automatic) pushing of all connections to remote system.
    push:
       remote:
           prod:
               name: "OpenConext Engineblock"
               url: "http://janus:janus_password@api.myconext.org/api/connections"
       # Disable SSL verification for test environments (should not be on in prod!)
       requestOptions:
           verify: false
```
To make this push mechanism work, the server should be able to find itself as api.myconext.org, so add a line to the /etc/hosts file.
```
127.0.1.1	myconext.org myconext admin.myconext.org api.myconext.org
```
Test the push interface using curl
```
# curl -u janus:janus_password http://api.myconext.org/api/connections
```
The answer should indeed be "No body" (which proves succesful authentication for now).

Now you should be able to "Push" Service Registry metadata configuration to Engineblock.

Bootstrap the engineblock Janus API user
```
# mysql janus_db -e "INSERT INTO janus__user (userid, type, active, secret) VALUES ('engineblock', 'a:2:{i:0;s:9:\"technical\";i:1;s:5:\"admin\";}', 'yes', 'eb_password');"
```

## Test
Testing requires engineblock debug mode to be disabled as it tries to validate any XML document that enginebock creates using only xsd's. This results in time-outs that break engineblock functionality.
```
# vi /etc/openconext/engineblock.ini
...
debug=false
```
If you want to keep debug=true enabled, you need to download the required xsd schema's and add them to the system libxml2 catalog
```
# vi /etc/xml/catalog
<catalog ...>
...
        <system systemId="http://www.w3.org/TR/2002/REC-xmldsig-core-20020212/xmldsig-core-schema.xsd"
                uri="file:///opt/OpenConext/engineblock/xsd/xmldsig-core-schema.xsd"/>
        <system systemId="http://www.w3.org/TR/2002/REC-xmlenc-core-20021210/xenc-schema.xsd"
                uri="file:///opt/OpenConext/engineblock/xsd/xenc-schema.xsd"/>
        <system systemId="http://docs.oasis-open.org/security/saml/v2.0/saml-schema-assertion-2.0.xsd"
                uri="file:///opt/OpenConext/engineblock/xsd/saml-schema-assertion-2.0.xsd"/>
        <system systemId="http://www.w3.org/2001/xml.xsd"
                uri="file:///opt/OpenConext/engineblock/xsd/xml.xsd"/>
</catalog>
```
... and download all mentioned xsd's to /opt/www/engineblock/xsd using wget
```
# cd /opt/OpenConext/engineblock
# mkdir xsd; cd xsd
# wget http://docs.oasis-open.org/security/saml/v2.0/saml-schema-metadata-2.0.xsd
# wget http://docs.oasis-open.org/security/saml/v2.0/saml-schema-assertion-2.0.xsd
# wget http://www.w3.org/TR/2002/REC-xmldsig-core-20020212/xmldsig-core-schema.xsd
# wget http://www.w3.org/TR/2002/REC-xmlenc-core-20021210/xenc-schema.xsd
# wget https://www.w3.org/2001/xml.xsd
```
Engineblock will now validate it's own metadata against locally cached xsd's

Open http://engine.myconext.org/ en click on the following predefined links
```
http://engine.myconext.org/authentication/idp/metadata
http://engine.myconext.org/authentication/sp/metadata
```
This should result in correct IdP and SP XML metadata files.

## Test using Profile and debug pages
Add a (test) IdP en the Profile test-SP to Service Registry metadata store in Janus

For the Profile test-SP save the contents of the xml document in location http://profile.myconext.org/simplesaml/module.php/saml/sp/metadata.php/default-sp and import into Service Registry as Service Provider.
Don't forget to switch entity ID to production status, assign at least one IdP (default is all) and push configuration to engineblock.

Disable secure cookies
```
# vi engineblock/vendor/simplesamlphp/simplesamlphp/config/config.php
-        'session.cookie.secure' => TRUE,
+        'session.cookie.secure' => false,
```
Now both http://profile.myconext.org/ and http://engine.myconext.org/authentication/sp/debug can be tested after the addresses are made resolvable from the hosts's environment.
