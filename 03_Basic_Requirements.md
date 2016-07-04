# Basic installation requirements
In this chapter we will start with the basic requirements to start installing OpenConext. Later on in the how-to there will be a couple of extra requirements installed, but I keep them there so that it is obvious they serve that particular part of the installation.

I am aware most, if not all Debian/Ubuntu installation manuals use the sudo commandline standard to execute commands with elevated root privileges. Throughout the whole of this manual however I became root once and executed all commands in a root shell for convenience.

## Become root
```
$ sudo su -
[sudo] password for user:
# _
```

## Apache, PHP, MariaDB, Memcached and git
Start with an up-to-date OS
```
# apt-get update
# apt-get upgrade
```
Install Apache package and PHP module
```
# apt-get install apache2 libapache2-mod-php
```

Install PHP dependancies
```
# apt-get install php-curl php-gmp php-mysql php-memcache php-mcrypt php-gd php-simplexml php-zip
```

Install Memcached
```
# apt-get install memcached
```

Install MariaDB
```
# apt-get install mariadb-common mariadb-client mariadb-server
```

And last, install commandline tools git, curl and jq (jq is nice tool to pretty-print jason output).
```
# apt-get install git-core curl jq
```

We're now ready to begin our journey.

Next chapter: [Service Registry](https://github.com/mrvanes/OpenConext-Howto/blob/master/04_ServiceRegistry.md)
