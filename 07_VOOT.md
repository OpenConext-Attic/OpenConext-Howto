# VOOT
VOOT is a group resource API translation layer and stands for Virtual Organisation Orthogonal Technology. Engineblock can only speak VOOT and VOOT translates Grouper API to VOOT for Engineblock.

VOOT is a java application and is built with maven. It needs java 8 so I'm happy Grouper runs on 8 as well (see previous Chapter).

Install requirements
```
# apt-get install openjdk-8-jre maven
```
Clone, build and install VOOT
```
# cd /opt/OpenConext
# git clone https://github.com/mrvanes/OpenConext-voot.git
# cd OpenConext-voot
```
Configure an external provider for grouper
```
# vi src/main/resources/externalProviders.yml
```
```
externalGroupProviders:
  - {
      type: "grouper",
      url: "http://localhost:8080/grouper-ws/services/GrouperService_v2_0",
      credentials: {
        username: "GrouperSystem",
        secret: "admin"
      },
      schacHomeOrganization: "myconext.org",
      name: "OpenConext",
      timeoutMillis: 2000
     }
```
```
# vi src/main/resources/application.properties
```
```
server.port=9090

externalProviders.config.path = file:/opt/OpenConext/OpenConext-voot/externalProviders.yml

# Details needed so that we may check tokens presented to us by clients. This application uses them to authenticate via
# Basic authentication with the oAuth server.
authz.checkToken.endpoint.url=http://localhost:8081/oauth/check_token
authz.checkToken.clientId=vootservice
authz.checkToken.secret=secret

oidc.checkToken.endpoint.url=http://localhost:8081/introspect
oidc.checkToken.clientId=https@//oidc.localhost.myconext.org
oidc.checkToken.secret=secret

checkToken.cache=true
checkToken.cache.duration.milliSeconds=300000

endpoints.enabled=false
endpoints.health.enabled=true
endpoints.info.enabled=true

info.build.artifact=@project.artifactId@
info.build.version=@project.version@

logging.level.voot: DEBUG
logging.level.org.springframework.security: DEBUG
```
Build OpenConext-voot, link the result and copy properties and provider files to the working directory
```
# mvn package -D skipTests
# ln -s target/voot-service-1.6.2.jar voot-service.jar
# cp src/main/resources/application.properties .
# cp ./src/main/resources/externalProviders.yml .
```
Create a systemd startup service
```
# vi /etc/systemd/system/voot.service
```
```
[Unit]
Description=VOOT Service

[Service]
Type=simple
User=root
ExecStart=-/usr/bin/java -Xms128m -Xmx128m -jar /opt/OpenConext/OpenConext-voot/voot-service.jar
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
Activate and start voot.service
```
# systemctl enable voot.service
# systemctl start voot.service
# tail -f /var/log/syslog
# systemctl status voot.service
```
Test http://localhost:9090/
```
# curl -s http://localhost:9090/ | jq .
{
  "error": "unauthorized",
  "error_description": "Full authentication is required to access this resource"
}
```
This oAuth error is OK and will be solved after installing oAuth in the next Chapter (OpenConext is quite entangled).

Next chapter: [Authz](https://github.com/mrvanes/OpenConext-Howto/blob/master/08_Authz.md)
