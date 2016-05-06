# Service Registry (Janus)
Service Registry is the central (SAML) metadata storage facility of the OpenConext suite. Without ServiceRegistry, Engineblock won't work.
By default all tools will be installed under the /opt/OpenConext application root.
```
# mkdir -p /opt/OpenConext
# cd /opt/OpenConext
```
Service Registry is historically distributed as a SimpleSAMLphp module and the only part of OpenConext that can't be found under the OpenConext github account.
We start out with a basic simpleSAMLphp installation and call this serviceregistry for convenience. Make sure a correct admin login is setup.
```
# wget https://simplesamlphp.org/res/downloads/simplesamlphp-1.14.1.tar.gz
# tar -zxf simplesamlphp-1.14.1.tar.gz
# ln -s  simplesamlphp-1.14.1 serviceregistry
# cd  serviceregistry
```
Create a hashed admin password
```
# bin/pwgen.php
```
Put the result of this command in the config.php and set memcache to be the default store type.
```
# vi config/config.php
-  'baseurlpath' => 'simplesaml/',
+  'baseurlpath' => 'serviceregistry/',
...
-  'auth.adminpassword' 	=> '123',
+  'auth.adminpassword' 	=> '{SSHA256}......',
...
-  'secretsalt' => 'defaultsecretsalt',
+  'secretsalt' => <follow the instructions above this setting>,

- 'store.type'			=> 'phpsession',
+ 'store.type'			=> 'memcache',
...
   'memcache_store.prefix' => "myconext",
```
Make a git clone of Janus in the module directory
```
# cd modules
# git clone https://github.com/mrvanes/janus.git
# cd janus
```
Janus doesn't come with it's own composer.phar in the bin directory, so we download our own.
```
# cd bin
# curl -sS https://getcomposer.org/installer | php
# cd ..
```
Create an empty database for Janus. All database examples in this howto assume a passwordless login for root (MariaDB default).
```
# mysql
> CREATE DATABASE janus_db;
> CREATE USER 'janus'@'localhost' IDENTIFIED BY 'janus_password';
> GRANT ALL PRIVILEGES ON janus_db.* TO 'janus'@'localhost' WITH GRANT OPTION;
> FLUSH PRIVILEGES;
> quit
```
Bootstrap a default configuration
```
# cp app/config-dist/config_janus_core.yml app/config
```
Now we can use composer.phar to download and install the dependancies.
```
# bin/composer.phar install
```
Janus installation will ask some questions to bootsrap a basic configuration. For the required secret you can use the following command:
```
# tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 	2>/dev/null;echo
```
Execute the migrate.sh script to populate the Janus database, ignore the errors.
```
# bin/migrate.sh
```
For authentication Janus relies on any user simpleSAMLphp can provide through it's authentication mechanism. Normally this is done by bootstrapping an IdP in the parent simpleSAMLphp installation which would require a working Engineblock. This creates a circular dependancy which we will shortcut by assigning the default admin user with password we created a couple of lines above as default admin user for Janus.
```
# mysql janus_db -e "INSERT INTO janus__user (userid, type, active) VALUES ('admin', 'a:2:{i:0;s:9:\"technical\";i:1;s:5:\"admin\";}', 'yes');"
```
We will use the 'user' attribute in Janus as link populated by the simpleSAMLphp admin authentication plugin.
```
# vi ./app/config/config_janus_core.yml
 auth: 'admin'
 ...
 useridattr: 'user'
```
Make sure apache user is owner of the serviceregistry folder(s)
```
# chown -R www-data. /opt/OpenConext/serviceregistry
```
Create Janus cache and log directories
```
# mkdir -p /var/cache/janus-ssp/janus
# chown www-data: /var/cache/janus-ssp/janus
# mkdir -p /var/log/janus-ssp/janus
# chown www-data: /var/log/janus-ssp/janus
```
Carefully inspect and edit app/config/config_janus_core.yml. I strongly reduced the number of workflow states to start with:
```
    # Configuration of systems in JANUS
    workflowstates:
        testaccepted:
            name:
                en: Test
            description:
                en: 'All test should be performed in this state'
            isDeployable: true
            textColor: red
            abbr: 'TA'
        prodaccepted:
            name:
                en: Production
            description:
                en: 'The connection is on the Production system'
            isDeployable: true
            textColor: green
            abbr: 'PA'
[...]
    workflow:
        testaccepted:
            prodaccepted:
                role:
                    - operations
                    - technical
                    - secretariat
        prodaccepted:
            testaccepted:
                role:
                    - operations
                    - technical
                    - secretariat
```
Ubuntu typically stores ca-certificates bundle in a different place
```
    # CA bundle used for checking,
    # by default check for path used by ca-certificates package
    ca_bundle_file: /etc/ssl/certs/ca-certificates.crt
```
Create an apache virtualhost for service registry
```
# vi /etc/apache2/sites-available/serviceregistry.conf
<VirtualHost *:80>
        ServerName serviceregistry.myconext.org
        DocumentRoot /var/www/html

        RewriteEngine On
        RewriteRule ^/$ /serviceregistry/module.php/janus/index.php [L,R=301]

        Alias /serviceregistry /opt/OpenConext/serviceregistry/www
        Alias /janus /opt/OpenConext/serviceregistry/modules/janus/web
</VirtualHost>
```
Adjust the global apache conf to allow for documents to be served from the /opt/OpenConext top directory.
```
# vi /etc/apache2/apache2.conf
<Directory /var/www/> 
       Options Indexes FollowSymLinks 
       AllowOverride None 
       Require all granted 
</Directory>
+
+ <Directory /opt/OpenConext/> 
+       Options Indexes FollowSymLinks 
+       AllowOverride None 
+       Require all granted 
+ </Directory>
```
Activate the  apache rewrite module, the serviceregistry vhost and rerstart apache
```
# a2enmod rewrite
# a2ensite serviceregistry
# service apache2 restart
```
The migrate scripte executed earlier takes ownership of the cache and log directories. Assign them to apache user
```
# chown -R www-data: /var/cache/janus-ssp/janus
# chown -R www-data: /var/log/janus-ssp/janus
```
Make sure the vhost is reachable under it's chosen hostname by editing the hosts's hostfile or configure DNS appropriately.
```
# vi /etc/hosts
x.x.x.x		serviceregistry.myconext.org
```
Service Registry should now be available on http://serviceregistry.myconext.org/serviceregistry/module.php/janus/index.php
