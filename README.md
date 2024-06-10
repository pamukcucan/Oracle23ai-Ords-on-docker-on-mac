# Oracle23ai-Ords-on-docker-on-mac
How to set up dockerized Oracle23 &amp; Ords environment on Mac

1) Install homebrew (check if it is already installed with: $brew --version)
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

2) install colima version 0.5.6: -- newer version has some problem -- re-check this
   (https://github.com/abiosoft/colima/releases)

$ sudo mkdir -p /usr/local/bin
$ sudo curl -L -o /usr/local/bin/colima https://github.com/abiosoft/colima/releases/download/v0.5.6/colima-Darwin-arm64 && sudo chmod +x /usr/local/bin/colima
$ ln -s /usr/local/bin/colima /opt/homebrew/bin/colima

3) install docker
$ brew install docker

$brew --version
Homebrew 4.2.15
$ colima --version
colima version v0.5.6
$ docker --version
Docker version 26.0.0, build 2ae903e86c

4) brew install lima


5) start colima

$ colima start \
    --arch x86_64 \
    --vm-type=vz \
    --vz-rosetta \
    --mount-type=virtiofs \
    --memory 8
INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0000] creating and starting ...                     context=vm
INFO[0051] provisioning ...                              context=docker
INFO[0051] starting ...                                  context=docker
INFO[0058] done


$ colima status
INFO[0000] colima is running using macOS Virtualization.Framework
INFO[0000] arch: x86_64                                
INFO[0000] runtime: docker                              
INFO[0000] mountType: virtiofs                          
INFO[0000] socket: unix:///Users/AIKINCI/.colima/default/docker.sock

$ colima list 
PROFILE    STATUS     ARCH      CPUS    MEMORY    DISK     RUNTIME    ADDRESS
default    Running    x86_64    2       8GiB      60GiB    docker

#BU LİSTEDE TEK COLİMA OLMASI KRİTİK

$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES


$ docker images

$ docker context show
colima

$ docker pull container-registry.oracle.com/database/free

$ docker images
REPOSITORY                                    TAG       IMAGE ID       CREATED        SIZE
container-registry.oracle.com/database/free   latest    39cabc8e6db0   6 months ago   9.16GB

docker volume ls

docker volume create oradata


$ docker image inspect container-registry.oracle.com/database/free | grep INSTALL_FILE_1
                "INSTALL_FILE_1=oracle-database-free-23c-1.0-1.el8.x86_64.rpm",

$ docker run -d \
  --name ora23c \
  -p 1522:1521 \
  --mount source=oradata,target=/opt/oracle/oradata \
  container-registry.oracle.com/database/free

$ docker ps
docker ps
CONTAINER ID   IMAGE                                         COMMAND                  CREATED              STATUS                             PORTS                                       NAMES
d0dbb49bdd0f   container-registry.oracle.com/database/free   "/bin/bash -c $ORACL…"   About a minute ago   Up 25 seconds (health: starting)   0.0.0.0:1522->1521/tcp, :::1522->1521/tcp   ora23c

#Burada dışardan docker a gelen istekler 1522 üzerinden geliyor, DB imajına da 1521 üzerinden gidiyor.

$ docker logs -f 'containerID'

Starting Oracle Net Listener.
Oracle Net Listener started.
Starting Oracle Database instance FREE.
Oracle Database instance FREE started.

The Oracle base remains unchanged with value /opt/oracle
#########################
DATABASE IS READY TO USE!
#########################


$ docker exec -it ora23c ./setPassword.sh oracle

docker exec -it ora23c /bin/bash

sqlplus / as sysdba

#bağlandığını gördük

exit

lsnrctl status

LSNRCTL for Linux: Version 23.0.0.0.0 - Production on 08-JUN-2024 16:06:13

Copyright (c) 1991, 2024, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC_FOR_FREE)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 23.0.0.0.0 - Production
Start Date                08-JUN-2024 15:59:52
Uptime                    0 days 0 hr. 6 min. 21 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Default Service           FREE
Listener Parameter File   /opt/oracle/product/23ai/dbhomeFree/network/admin/listener.ora
Listener Log File         /opt/oracle/diag/tnslsnr/d1b0f7fc8c00/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC_FOR_FREE)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
Services Summary...
Service "16df542da83f091ce0630500580a27e7" has 1 instance(s).
  Instance "FREE", status READY, has 1 handler(s) for this service...
Service "FREE" has 1 instance(s).
  Instance "FREE", status READY, has 1 handler(s) for this service...
Service "FREEXDB" has 1 instance(s).
  Instance "FREE", status READY, has 1 handler(s) for this service...
Service "PLSExtProc" has 1 instance(s).
  Instance "PLSExtProc", status UNKNOWN, has 1 handler(s) for this service...
Service "freepdb1" has 1 instance(s).
  Instance "FREE", status READY, has 1 handler(s) for this service...
The command completed successfully


#SQL DEVELOPER İLE BAĞLANMAK İÇİN:

Connection name: ora23c
Authentication type: default
Role: SYSDBA
Username: SYS
Password: oracle  >> yukarda belirledik
save psw
Connection Type: Basic
Hostname: localhost
Port:1522
Type: Service Name
Service Name: freepdb1 


INSTALL ORDS

docker pull container-registry.oracle.com/database/ords:latest

#DB e bağlan ve aşağıdaki user ve grantleri ver

create user ords identified BY oracle;

grant resource, CONNECT, SYSDBA to ords;

alter user ords quota unlimited on users;

exit

cd home

mkdir ords_secrets
mkdir ords_config

cd ords_secrets

vi conn_string

CONN_STRING=ords/oracle@172.17.0.1:1522/freepdb1

#Port and instance depends on your configuration

docker run -d --name ords -v `pwd`/ords_secrets/:/opt/oracle/variables -v `pwd`/ords_config/:/etc/ords/config/  -p 8181:8181 -p 27017:27017 container-registry.oracle.com/database/ords:latest


#Apex hatası alırsan aşağıdaki ile tekrar dene


docker run -d -e IGNORE_APEX=TRUE --name ords -v `pwd`/ords_secrets/:/opt/oracle/variables -v `pwd`/ords_config/:/etc/ords/config/  -p 8181:8181 -p 27017:27017 container-registry.oracle.com/database/ords:latest


























