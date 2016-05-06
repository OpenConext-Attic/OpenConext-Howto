# Introduction

OpenConext is developed on CentOS 6, soon 7 and the easy install requires a compatible setup on the target server. This manual however explores the rough journey of installing all the required parts on a fresh Ubuntu Xenial 16.04 LTS server. The purpose of the excercise being two-fold: help those unwilling or unable to fullfil the CentOS requirements and provide a thorough understanding of the actions "magically" done by the Ansible installation by executing them step-by-step, by hand.

Please beware I am not one of the core developers of OpenConext (although I did have some looks at the code here and there) so everything I have writtern in this document is a best effort of understanding the requirements and human interpretation of the Ansible installation, translated to the Ubuntu (Debian) packages and commands to provide an as OS agnostic installation as possible.

My ultimate goal while writing this how-to was profoundly understanding the mutual dependancies of the parts.

Most of the installations still point to my forks of the code because I hit some small snags here and there and at that time needed to make small adjustments to make the code or configurations work on a Debian based installation. I would recommend anyone to first try to checkout the original repositories so that you work with a recent codebase where possible and fallback to my forks if needed. I let the finding of those repositories as an excersise to the reader as it will greatly help in deeper understanding of the structure of OpenConext.

Since understanding OpenConext and it's mutual dependancies was my main goal, I decided to strip out all secured access to any http endpoint. Configuring secured URL's isn't hard (or shouldn't be) but significantly and unnecessarily clogs the instructions in this how-to, while making debugging of problems harder. Make sure to secure all endpoints before putting OpenConext into production.

## Hostnames
These are the (fictual) hostnames I used during installation of OpenConext. They only exist in my local hosts files, both on the host I work on and in the VM I used to install OpenConext. I deliberately departed from the default hostname openconext.org so that any deviation in hostname would be visible in all respective configuration files.
```
myconext.org                  Node
serviceregistry.myconext.org  Service Registry vhost
auth.myconext.org             Engineblock SAML vhost
api.myconext.org              Engineblock API vhost
profile.myconext.org          Engineblock Profile SP vhost
static.myconext.org           Engineblock static content (eg IdP icons) vhost

sp.myconext.org               Dummy Service Provider
idp.myconext.org              Dummy Identity Provider

myconext.org:8082             Grouper
voot.myconext.org             Voot vhost
server.authz.myconext.org     oAuth server
admin.authz.myconext.org      oAuth admin interface
play.authz.myconext.org       oAuth playground interface

pdp.myconext.org              Policy Decision Endpoint Admin GUI
```
## OpenConext Dependancy tree
This picture tries explain the mutual dependancies of the OpenConext suite. The hard requirement for LDAP is being removed for version 5 of Engineblock. This is a welcome change for anyone only wanting to install the core components Service Registry and Engineblock, but hampers the further installation of Grouper, so I leave it in for the time being (and because Engineblock 5 is not yet released).

* Apache (mod-php7)
* PHP (curl, gmp, mysql, memcache, mcrypt, gd, ldap)
* Memcached
* MySQL (MariaDB)
* Git
  * Service Registry (Janus)
  * LDAP
    * EngineBlock
    * Grouper
      * Voot
        *	Authorization-server
          *	Authorization-admin
          * Authorization-playground
        * PDP
          * PDP-GUI
  			
