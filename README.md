#Signal Community Edition Installation Steps
Author: Luca Conte luca@riseup.net
last edit: 2016.03.27 ("now" in the following text)

##Abstract
This paper woluld be a quickstart for anyone aims to setup a working installation of OpenWhisperSystems (OWS) Signal software either the backend part (Server) and the frontend one (Android, iOS and Chrome/Chromium extention).

##What Is Signal
Signal is a messaging software (aka App) like, for example, the well known WhatsApp (or Telegram) built with security and privacy in mind. It implements an Ent-To-End encryption (Curve25519, AES-256 and HMAC-SHA256)[1] and it woluld represent (and probably *do*)  *[...]the current state of the art in private messaging* (as OWS says[1]).


##The server part
The original server components could be found here:
https://github.com/WhisperSystems/TextSecure-Server
and here:
https://github.com/WhisperSystems/PushServer

"component*s*" and not "component" because actually the OWS Signal backend is an ecosystem builded over two (technically) decoupeld "players": TextSecure-Server (TSS) which expose the API used by clients and a PushServer (PS) which perform APN and GCM notifications. In reality there is/was a third player, the RedPhoneServer, responsible for VOIP Crypted communications, but its sources are nota avaiable so every functionality relate to it at the moment **must** be disabled (see later).

Due to some trick that anyone need to perform in order to han a working server I've forked some OWS repositories and pushed my local modifications into them. So If you are impatient it's better for you perform clone (or fork) operations from my forks:
https://github.com/lucaconte/TextSecure-Server
and here:
https://github.com/lucaconte/PushServer

###Prerequisites
- Java  1.7. This tutorial is based on the Oracle version, may be interesting perform these steps using an OpenJdk.
- A linux box. I've used a Debian testing environment for bulding and debug purposes and a Debian Jessie as production environment.
- A working [Redis](http://redis.io "Redis") (for caching purposes)
- A [PostgreSQL](http://www.postgresql.org/) instance
- A Twillio account for SMS registration process confirmation
- An Amazon AWS S3 bucket service for messages' attachments
- A GCM account from google developer console: a senderId add a ApiKey. Create a project, senderId is the *projectNumber*. For ApiKey, go to API management -> credentials -> create credentials -> API Keys -> Server Key and save this key 
- An APN account for Apple Push Notifications (two pem certificates) [ask]

###Let's start with the code
Due to some trick that anyone need to perform in order to have a working server I've forked some OWS repositories and pushed my local modifications into them. So If you are impatient it's better for you perform clone (or fork) operations from my forks:
https://github.com/lucaconte/TextSecure-Server
and here:
https://github.com/lucaconte/PushServer

Than
    git clone --depth 1 https://github.com/lucaconte/TextSecure-Server.git
and 
    git clone --depth 1 https://github.com/lucaconte/PushServer.git


Now start with compiling PushServer:
   cd PushServer
   mvn clean install


If you try now  to perform a build of TSS You obtain a broken dependency error related to WebSocked-Resources project (also forked by me):

   [ERROR] Failed to execute goal on project TextSecureServer: Could not resolve dependencies for project org.whispersystems.textsecure:TextSecureServer:jar:0.92: Failed to collect dependencies at org.whispersystems:websocket-resources:jar:0.3.2: Failed to read artifact descriptor for org.whispersystems:websocket-resources:jar:0.3.2: Could not find artifact org.whispersystems:parent:pom:0.3.2 in gcm-server-repository (https://raw.github.com/whispersystems/maven/master/gcm-server/releases/) -> [Help 1]

To bypass this error clnoe WebSocket-Resource separately:

   git clone --depth 1 https://github.com/lucaconte/WebSocked-Resources.git

build it:

   cd ../WebSocker-Resource
   mvn clean install

It will fail with an error relate to JavaDoc:
   [ERROR] Failed to execute goal org.apache.maven.plugins:maven-javadoc-plugin:2.8.1:jar (attach-javadocs) on project websocket-resources: MavenReportException: Error while creating archive:
   [ERROR] Exit code: 1 - javadoc: error - invalid flag: -Xdoclint:none

**but** anyway the artifact we need has been builded: ./library/targer/websocket-resources-0.3.2.jar
Now it's necessary to install it into our local Maven repository with correct "names":
    mvn install:install-file -Dfile=./library/target/websocket-resources-0.3.2.jar -DgroupId=org.whispersystems -DartifactId=websocket-resources -Dversion=0.3.2 -Dpackaging=jar

After this finally we are able to compile TSS
    cd ..
    cd TextSecure-Server
    mvn install

Troubleshootinh: if you have some problem in building related to "tests" add
    -DskipTests

###Run the servers
Configurations files provided by OWS in yml format **doesn't work**! Mainly for two reasons:
1 They are empty and pratically every param inside them is mandatory
2 (heavier) Some fields are mispelled.

Here a working TextSecure server file filled with **fake** values you have to provide your own values:


    \# This is the sample config/production.yml file for the TextSecure Server
    \# Pay attention! To start TextSecur server you will need to install and start PushServer
    
    twilio: 
      accountId: AZ8acwec7fc5eaf209a69f05e50a8b2675 \#fake
      accountToken: 556658d1458f8852ed183a64c40ec5d1 \#fake
      numbers:
        -
          +33756796138 \#fake
      localDomain: foo.org
    
    push:
      host: localhost
      port: 9090
      username: 123
      password: 123
    
    s3:
       accessKey: ABCDEFGCUFYDHVM2LXXX \#fake
       accessSecret: W0UfGDddfAbqYyCTIIbSQlDtreTGokOs0OTpL0SE \#fake
       attachmentsBucket: thenameofyouts3buket \#fake
    
    directory:
      url: "redis://localhost:6379/0"
    
    cache:
      url: "redis://localhost:6379/1"
    
    server:
      applicationConnectors:
        - type: http
          port: 8080
          \#keyStorePath: config/example.keystore
          \#keyStorePassword: whisper
          \#validateCerts: true
      adminConnectors:
        - type: http
          port: 8081
          \#keyStorePath: config/example.keystore
          \#keyStorePassword: whisper
          \#validateCerts: true
    
    
    websocket:
      enabled: true
    
    messageStore: \# Postgres database configuration for message store
      driverClass: org.postgresql.Driver
      user: "signal"
      password: "thepassword"
      url: "jdbc:postgresql://localhost:5432/messagedb"
    
    database:
      driverClass: org.postgresql.Driver
      user: "signal"
      password: "thepassword"
      url: "jdbc:postgresql://localhost:5432/accountsdb"
      properties:
        charSet: UTF-8
    
    \#federation: \# is disabled
    
    logging:
      level: INFO
      appenders:
        - type: file
          currentLogFilename: /tmp/textsecureshserver.log
          archivedLogFilenamePattern: /temp/textsecureserver-%d.log.gz
          archivedFileCount: 5
        - type: console
    
    
    redphone:
        authKey: 1234567890 \#fake


---
[1] http://support.whispersystems.org/hc/en-us/articles/212477768
[2] Originally https://github.com/WhisperSystems/TextSecure-Server/issues/44 but it has been removed 

