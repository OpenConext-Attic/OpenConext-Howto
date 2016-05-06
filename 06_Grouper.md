# Grouper
Although not a direct requirement of OpenConext all remaining parts require user's adminstrator virtual organisation membership of some sort that is consumed from the VOOT (Virtual Organisation Orgthogonal Technology) API (wich is described in the next chapter). VOOT can rely on Grouper for (admin) groupmembership claims. Grouper has a nice admin interface that makes it easy to assign test users to test admin organisations where needed in the rest of the How-to's.

To smoothly run Grouper your VM needs to be equiped with at least 1.5G of memory. Although the preferred Java version is 7, it can be made to run on 8 which is the only choice (from packages) in Xenial.

First, install openjdk-8 and dos2unix
```
# apt-get install openjdk-8-jdk dos2unix
```
Below is a transcript of my Grouper install. Please inspect my answers and adjust accordingly to cater for local differences.
```
# cd /var/tmp
# wet http://software.internet2.edu/grouper/release/2.3.0/grouper.installer-2.3.0.tar.gz
# tar -zxf grouper.installer-2.3.0.tar.gz
# mkdir -p /opt/OpenConext/Grouper
# cd /opt/OpenConext/Grouper
# java -jar /var/tmp/grouper.installer-2.3.0/grouperInstaller.jar
```
You can proceed with ```<enter>``` except where the required answer is not the default
```
... [install]: <enter>
... [/opt/OpenConext/Grouper]: <enter>
... [0.0.0.0]: <enter>
Installing grouper version: 2.2.2
Downloading ...
... executable (t|f)? [t]: <enter>
... dos2unix on gsh.sh (t|f)? [t]: <enter>
... included hsqldb database (t|f)? [t]: <enter>
... able to start) (t|f)? [t]: <enter>
.... all patches (t|f)? [t]: <enter>
... <enter> to continue... <enter>
... progress of SQL scripts) (t|f)?<enter>
... tables, add new ones) (t|f)? t<enter>
... quickstart subjects to DB (t|f)? [t]: f<enter>
... quickstart data to registry (t|f)? [t] f<enter>
... tomcat memory limit (t|f)? [t]: <enter>
... set tomcat scripts to executable (t|f)? [t]: <enter>
... dos2unix on tomcat sh files (t|f)? [t]: <enter>
... tomcat to run on (HTTP, JK, shutdown): [8080, 8009, 8005]: <enter>
... to UTF-8 in tomcat server.xml <Connector> elements (t|f)? [t]: <enter>
... log dir of UI (t|f)? [t]:f <enter>
... URL path for the UI [grouper]: <enter>
Enter the GrouperSystem password: admin
... set the GrouperSystem password in ... [t]: <enter>
... see if tomcat was able to start (t|f)? [t]: <enter>
... do you want it rebuilt? (t|f) [t]: <enter>
... if tomcat was able to stop (t|f)? [t]: <enter>
... set the log dir of WS (t|f)? [t]: f<enter>
... URL path for the WS [grouper-ws]: <enter>
... stop tomcat anyway? (t|f)? [f] <enter>
... see if tomcat was able to start (t|f)? [t]: <enter>
... install the provisioning service provider (t|f)? [t]: <enter>
... check ps -ef | grep gsh | grep loader) (t|f)? [f]: <enter>
```
Check grouper at the URL http://myconext.org:8080/grouper, user: GrouperSystem, pwd: admin

Starting and stopping requires two daemons (hsqldb service and tomcat, so I created one single start script)
```
# cd /opt/OpenConext/Grouper
# vi startup.sh
```
```
#!/bin/sh
export JAVA_HOME=/usr 
cd /opt/OpenConext/Grouper/grouper.apiBinary-2.3.0
$JAVA_HOME/bin/java -cp lib/jdbcSamples/hsqldb.jar org.hsqldb.Server -database.0 file:grouper -dbname.0 grouper -port 9001 & 
sleep 5 
cd ../apache-tomcat-6.0.35/ 
bin/startup.sh 
cd ..
```
Make startup.sh executable
```
# chmod 755 startup.sh
```
Create shutdown script
```
# vi shutdown.sh
```
```
#!/bin/sh
pkill -f hsqldb.jar
pkill -f bootstrap.jar
```
Make shutdown.sh executable
```
# chmod 755 shutdown.sh
```
Add LDAP as entity source, please note the dependancies to your own directory server.
We will reuse identities created by Engineblock for consent as the entity source. This means the entities will only be available after at least one succesful login through Engineblock!
```
# dos2unix grouper.ui-2.3.0/dist/grouper/WEB-INF/classes/sources.xml
# vi grouper.ui-2.3.0/dist/grouper/WEB-INF/classes/sources.xml
```
Find a suitable place to paste the suggested ```<source>``` child below. An suitable anchor should be the end of a previous ```</source```.
You can also adjust the existing example JndiSourceAdapter in place, which is what I did.
```
... Previous source anchor
  </source>
  <source adapterClass="edu.internet2.middleware.grouper.subj.GrouperJndiSourceAdapter">
    <id>openconext-ldap</id>
    <name>OpenConext LDAP</name>
    <type>person</type>
    <init-param>
      <param-name>INITIAL_CONTEXT_FACTORY</param-name>
      <param-value>com.sun.jndi.ldap.LdapCtxFactory</param-value>
    </init-param>
    <init-param>
      <param-name>PROVIDER_URL</param-name>
      <param-value>ldap://localhost:389</param-value>
    </init-param>
    <init-param>
      <param-name>SECURITY_AUTHENTICATION</param-name>
      <param-value>simple</param-value>
    </init-param>
    <init-param>
      <param-name>SECURITY_PRINCIPAL</param-name>
      <param-value>cn=admin,dc=myconext,dc=org</param-value>
    </init-param>
    <init-param>
      <param-name>SECURITY_CREDENTIALS</param-name>
      <param-value>admin</param-value>
    </init-param>
     <init-param>
      <param-name>SubjectID_AttributeType</param-name>
      <param-value>collabPersonId</param-value>
    </init-param>
     <init-param>
      <param-name>SubjectID_formatToLowerCase</param-name>
      <param-value>false</param-value>
    </init-param>
    <init-param>
      <param-name>Name_AttributeType</param-name>
      <param-value>cn</param-value>
    </init-param>
    <init-param>
      <param-name>Description_AttributeType</param-name>
      <param-value>cn</param-value>
    </init-param>
<!--
    <init-param>
      <param-name>VTLDAP_VALIDATOR</param-name>
      <param-value>ConnectLdapValidator|CompareLdapValidator</param-value>
    </init-param>
    <init-param>
      <param-name>VTLDAP_VALIDATOR_COMPARE_DN</param-name>
      <param-value>dc=myconext,dc=org</param-value>
    </init-param>
    <init-param>
      <param-name>VTLDAP_VALIDATOR_COMPARE_SEARCH_FILTER_STRING</param-name>
      <param-value>dc=myconext</param-value>
    </init-param>
-->   
    /// Scope Values can be: OBJECT_SCOPE, ONELEVEL_SCOPE, SUBTREE_SCOPE 
    /// For filter use 
    
    <search>
        <searchType>searchSubject</searchType>
        <param>
            <param-name>filter</param-name>
            <param-value>
                (&amp; (collabPersonId=%TERM%) (objectclass=collabPerson))
            </param-value>
        </param>
        <param>
            <param-name>scope</param-name>
            <param-value>
                SUBTREE_SCOPE            
            </param-value>
        </param>
        <param>
            <param-name>base</param-name>
            <param-value>
                dc=myconext,dc=org
            </param-value>
        </param>
         
    </search>
    <search>
        <searchType>searchSubjectByIdentifier</searchType>
        <param>
            <param-name>filter</param-name>
            <param-value>
                (&amp; (collabPersonId=%TERM%) (objectclass=collabPerson))
            </param-value>
        </param>
        <param>
            <param-name>scope</param-name>
            <param-value>
                SUBTREE_SCOPE            
            </param-value>
        </param>
        <param>
            <param-name>base</param-name>
            <param-value>
                dc=myconext,dc=org
            </param-value>
        </param>
    </search>
    
    <search>
       <searchType>search</searchType>
         <param>
            <param-name>filter</param-name>
            <param-value>
                (&amp; (|(collabPersonId=%TERM%)(cn=*%TERM%*)(mail=*%TERM%*))(objectclass=collabPerson))
            </param-value>
        </param>
        <param>
            <param-name>scope</param-name>
            <param-value>
                SUBTREE_SCOPE            
            </param-value>
        </param>
         <param>
            <param-name>base</param-name>
            <param-value>
                dc=myconext,dc=org
            </param-value>
        </param>
    </search>
    <init-param>
      <param-name>subjectVirtualAttribute_0_searchAttribute0</param-name>
      <param-value>${subjectUtils.defaultIfBlank(subject.getAttributeValueOrCommaSeparated('uid'), "")},${subjectUtils.defaultIfBlank(subject.getAttributeValueOrCommaSeparated('cn'), "")},${subjectUtils.defaultIfBlank(subject.getAttributeValueOrCommaSeparated('exampleEduRegId'), "")}</param-value>
    </init-param>
    <init-param>
      <param-name>sortAttribute0</param-name>
      <param-value>cn</param-value>
    </init-param>
    <init-param>
      <param-name>searchAttribute0</param-name>
      <param-value>searchAttribute0</param-value>
    </init-param>

     <!-- ##########################  STATUS SECTION for searches to filter out inactives and allow
                                                     the user to filter by status with e.g. status=all
                                                     this is optional, and advanced --> 
     <!-- column or attribute which represents the status -->
     <!--
     <init-param>
       <param-name>statusDatastoreFieldName</param-name>
       <param-value>status</param-value>
     </init-param> -->
     <!-- search string from user which represents the status.  e.g. status=active -->
     <!--
     <init-param>
       <param-name>statusLabel</param-name>
       <param-value>status</param-value>
     </init-param> -->
     <!-- available statuses from screen (if not specified, any will be allowed). comma separated list.
          Note, this is optional and you probably dont want to configure it, it is mostly necessary
          when you have multiple sources with statuses...  if someone types an invalid status
          and you have this configured, it will not filter by it -->
     <!-- 
     <init-param>
       <param-name>statusesFromUser<param-name>
       <param-value>Active, Inactive, Pending, All</param-value>
     </init-param> -->
     <!-- all label from the user -->
     <!--
     <init-param>
       <param-name>statusAllFromUser</param-name>
       <param-value>All</param-value>
     </init-param> -->
     <!-- if no status is specified, this will be used (e.g. for active only).  Note, the value should be of the
          form the user would type in -->
     <!-- 
     <init-param>
       <param-name>statusSearchDefault</param-name>
       <param-value>status=active</param-value>
     </init-param> -->
     <!-- translate between screen values of status, and the data store value.  Increment the 0 to 1, 2, etc for more translations.
          so the user could enter: status=active, and that could translate to status_col=A.  The 'user' is what the user types in,
          the 'datastore' is what is in the datastore.  The user part is not case-sensitive.  Note, this could be a many to one -->
     <!--
     <init-param>
       <param-name>statusTranslateUser0</param-name>
       <param-value>active</param-value>
     </init-param>
     <init-param>
       <param-name>statusTranslateDatastore0</param-name>
       <param-value>A</param-value>
     </init-param> -->
     <!-- ########################## END STATUS SECTION --> 


    <internal-attribute>searchAttribute0</internal-attribute>

    ///Attributes you would like to display when doing a search 
    <attribute>sn</attribute>
    <attribute>givenName</attribute>
    <attribute>mail</attribute>
    <attribute>displayName</attribute>
    <attribute>cn</attribute>
    <attribute>o</attribute>

    <!-- subject identifier to store in grouper's member table -->
    <init-param>
      <param-name>subjectIdentifierAttribute0</param-name>
      <param-value>collabPersonId</param-value>
    </init-param>
   
  </source>

  <source>
... Next source continues here
```
Grouper consist of multiple java components. They all need the same sources.xml. Copy or link to their respective directories.
```
# cp ./grouper.ui-2.3.0/dist/grouper/WEB-INF/classes/sources.xml ./grouper.apiBinary-2.3.0/conf/
# cp ./grouper.ui-2.3.0/dist/grouper/WEB-INF/classes/sources.xml ./grouper.ws-2.3.0/grouper-ws/build/dist/grouper-ws/WEB-INF/classes/
```
Restart Grouper with the new source defined
```
# ./shutdown.sh
# ./startup.sh
```
You may want to follow the startup sequence in apache-tomcat-6.0.35/logs/catalina.out to see if there are no problems.

Test grouper LDAP koppeling, first see if there is any usable engineblock consent user
```
# ldapsearch -H ldap://localhost -D "cn=admin,dc=myconext,dc=org" -w admin -b "dc=myconext,dc=org" collabPersonId
```
Look for an entry with valid collabPersonId, in the form of "urn:collab:person:*".

Visit Grouper's admin webinterface at http://myconext.org:8080/grouper and search (top right) for this collabPersonId.
When all went well, the LDAP entry should be found.

Also try the API (change REST url to reflect an existing collabPersonId)
```
# curl -u GrouperSystem:admin http://localhost:8080/grouper-ws/servicesRest/v1_3_000/subjects/urn:collab:person:myconext.org:student/groups
```
Expecting something along the lines of
```
{"WsGetGroupsLiteResult":{"responseMetadata":{"millis":"6828","serverVersion":"2.2.2"},"resultMetadata":{"resultCode":"SUCCESS","resultMessage":"Success for: clientVersion: 1.3.0, subjectLookups: Array size: 1: [0]: WsSubjectLookup[subjectId=urn:collab:person:mrvanes.com:S.tudent]\n\nmemberFilter: All, includeGroupDetail: false, actAsSubject: null\n, params: null\n fieldName1: null\n, scope: null, wsStemLookup: WsStemLookup[]\n, stemScope: null, enabled: null, pageSize: null, pageNumber: null, sortString: null, ascending: null\n, pointInTimeFrom: null, pointInTimeTo: null","success":"T"},"wsSubject":{"id":"urn:collab:person:mrvanes.com:S.tudent","name":"Stef Student","resultCode":"SUCCESS","sourceId":"openconext-ldap","success":"T"}}}
```
Optionally you can create systemd service scripts to start Grouper after reboot
```
# vi /etc/systemd/system/hsqldb.service
```
```
[Unit]
Description=Grouper HSQLDB Service

[Service]
Type=simple
User=root
WorkingDirectory=/opt/OpenConext/Grouper/grouper.apiBinary-2.3.0
ExecStart=/usr/lib/jvm/java-8-openjdk-amd64/bin/java -cp lib/jdbcSamples/hsqldb.jar org.hsqldb.Server -database.0 file:grouper -dbname.0 grouper -port 9001
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
```
# vi /etc/systemd/system/grouper.service
```
```
[Unit]
Description=Grouper Service
Requires=hsqldb.service
After=hsqldb.service

[Service]
Type=simple
User=root
Environment="JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64"
ExecStart=/opt/OpenConext/Grouper/apache-tomcat-6.0.35/bin/catalina.sh run
ExecStop=/bin/kill $MAINPID

[Install]
WantedBy=multi-user.target
```
Shutdown Grouper and then activate and start the services via systemd
```
# ./shutdown.sh
# systemctl enable hsqldb.service
# systemctl enable grouper.service
# systemctl start hsqldb.service
# systemctl start grouper.service
# tail -f /var/log/syslog
# systemctl status hsqldb.service
# systemctl status grouper.service
```
You can now proceed to the installation of the VOOT API for Grouper.
