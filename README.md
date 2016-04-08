#Signal Installation Steps
Author: Luca Conte luca@riseup.net

first draft: 20160827 ("now" in the following text)

released: 20160408

status: DRAFT

##Abstract
This paper would be a quickstart for anyone aims to setup a working installation of OpenWhisperSystems (OWS) Signal software either the backend part (Server) and the frontend one (Android, iOS and Chrome/Chromium extension).

##What Is Signal
Signal is a messaging software (aka App) like, for example, the well known WhatsApp (or Telegram) built with security and privacy in mind. It implements an Ent-To-End encryption \[[1](#one)\] (Curve25519, AES-256 and HMAC-SHA256) and it would represent (and probably *do*)  *[...]the current state of the art in private messaging* (as OWS says)\[[1](#one)\].


##The server part
The original server components could be found here:

    https://github.com/WhisperSystems/TextSecure-Server

and here:

    https://github.com/WhisperSystems/PushServer

"component*s*" and not "component" because actually the OWS Signal backend is an ecosystem built over two (technically) decoupled "players": TextSecure-Server (TSS) which expose the API used by clients and a PushServer (PS) which perform APN and GCM notifications. In reality there is/was a third player, the RedPhoneServer, responsible for VOIP Encrypted communications, but its sources are not available so every functionality relate to it at the moment **must** be disabled (see later).

Due to some trick that anyone need to perform in order to have a working server I've forked some OWS repositories and pushed my local modifications into them. So If you are impatient it's better for you perform clone (or fork) operations from my forks:

    https://github.com/lucaconte/TextSecure-Server

and here:

    https://github.com/lucaconte/PushServer

###Prerequisites
- Java  1.7. This tutorial is based on the Oracle version, may be interesting perform these steps using an OpenJdk.
- A linux box. I've used a Debian testing environment for building and debug purposes and a Debian Jessie as production environment.
- A working [Redis](http://redis.io "Redis") (for caching purposes)
- A [PostgreSQL](http://www.postgresql.org/) instance
- A Twillio account for SMS registration process confirmation
- An Amazon AWS S3 bucket service for messages' attachments
- A GCM account from google developer console: a senderId add a ApiKey. Create a project, senderId is the *projectNumber*. For ApiKey, go to API management -> credentials -> create credentials -> API Keys -> Server Key and save this key 
- An APN account for Apple Push Notifications (two pem certificates) [ask]

###Let's start with the code
Due to some tricks that anyone need to perform in order to have a working server I've forked some OWS repositories and pushed my local modifications into them. So If you are impatient it's better for you perform clone (or fork) operations from my forks:

    https://github.com/lucaconte/TextSecure-Server

and here:

    https://github.com/lucaconte/PushServer

Than

    git clone --depth 1 https://github.com/lucaconte/TextSecure-Server.git
and 

    git clone --depth 1 https://github.com/lucaconte/PushServer.git

Probably here you're interested in changing the APN invocation from production environment to sandbox one or vice versa. In this case go to "org.whispersystems.pushserver.senders.APNSender" class, method "public void start()" and edit this part uncommenting the right service's instantiation:

    //[LC] uncomment for production releases (with the right certificate)
    // this.pushApnService = APNS.newService()
    //                          .withCert(new ByteArrayInputStream(pushKeyStore), "insecure")
    //                         .asQueued()
    //                        .withProductionDestination().build();

    this.pushApnService = APNS.newService()
            .withCert(new ByteArrayInputStream(pushKeyStore), "insecure")
            .asQueued()
            .withSandboxDestination().build();



Now start with compiling PushServer:

    cd PushServer
    mvn clean install


If you try now  to perform a build of TSS You obtain a broken dependency error related to WebSocked-Resources project (also forked by me):

    [ERROR] Failed to execute goal on project TextSecureServer: Could not resolve dependencies for project org.whispersystems.textsecure:TextSecureServer:jar:0.92: Failed to collect dependencies at org.whispersystems:websocket-resources:jar:0.3.2: Failed to read artifact descriptor for org.whispersystems:websocket-resources:jar:0.3.2: Could not find artifact org.whispersystems:parent:pom:0.3.2 in gcm-server-repository (https://raw.github.com/whispersystems/maven/master/gcm-server/releases/) -> [Help 1]

To bypass this error clone WebSocket-Resource separately:

    git clone --depth 1 https://github.com/lucaconte/WebSocked-Resources.git

build it:

    cd ../WebSocker-Resource
    mvn clean install

It will fail with an error relate to JavaDoc:

    [ERROR] Failed to execute goal org.apache.maven.plugins:maven-javadoc-plugin:2.8.1:jar (attach-javadocs) on project websocket-resources: MavenReportException: Error while creating archive:
    [ERROR] Exit code: 1 - javadoc: error - invalid flag: -Xdoclint:none

**but** anyway the artifact we need has been built: ./library/targer/websocket-resources-0.3.2.jar
Now it's necessary to install it into our local Maven repository with correct "names":

    mvn install:install-file -Dfile=./library/target/websocket-resources-0.3.2.jar -DgroupId=org.whispersystems -DartifactId=websocket-resources -Dversion=0.3.2 -Dpackaging=jar

After this finally we are able to compile TSS

    cd ..
    cd TextSecure-Server
    mvn install

Troubleshooting: if you have some problem in building related to "tests" add
    -DskipTests

###The configuration files
Configurations files provided by OWS in yml format **doesn't work**! Mainly for two reasons:

1. They are empty and practically every param inside them is mandatory
2. (heavier) Some fields are misspelled.

Here a working TextSecure server file filled with **fake** values. You have to provide your own values:


    # This is the sample config/textsecure.yml file for the TextSecure Server
    # Pay attention! To start TextSecur server you will need to install and start PushServer
    
    twilio: 
      accountId: AZ8acwec7fc5eaf209a69f05e50a8b2675 #fake
      accountToken: 556658d1458f8852ed183a64c40ec5d1 #fake
      numbers:
        -
          +33756796138 #fake
      localDomain: foo.org
    
    push:
      host: localhost
      port: 9090
      username: 123
      password: 123
    
    s3:
       accessKey: ABCDEFGCUFYDHVM2LXXX #fake
       accessSecret: W0UfGDddfAbqYyCTIIbSQlDtreTGokOs0OTpL0SE #fake
       attachmentsBucket: thenameofyouts3buket #fake
    
    directory:
      url: "redis://localhost:6379/0"
    
    cache:
      url: "redis://localhost:6379/1"
    
    server:
      applicationConnectors:
        - type: http
          port: 8080
          #keyStorePath: config/example.keystore
          #keyStorePassword: example
          #validateCerts: true
      adminConnectors:
        - type: http
          port: 8081
          #keyStorePath: config/example.keystore
          #keyStorePassword: example
          #validateCerts: true
    
    
    websocket:
      enabled: true
    
    messageStore: # Postgres database configuration for message store
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
    
    #federation: # is disabled
    
    logging:
      level: INFO
      appenders:
        - type: file
          currentLogFilename: /tmp/textsecureshserver.log
          archivedLogFilenamePattern: /temp/textsecureserver-%d.log.gz
          archivedFileCount: 5
        - type: console
    
    
    redphone:
      authKey: 1234567890 #fake

Here a working PushServer server file filled with **fake** values. You have to provide your own values:

    redis:
      url: redis://localhost:6379/2
    
    authentication:
      servers:
        -
          name: 123
          password: 123
    gcm:
      xmpp: false
      apiKey: AIzaSyAsviyMy0kIe8uhCEfo8NbeqGoku7oOCi4 #fake
      senderId: 888789650296 #fake
      redphoneApiKey: AIddSyAsviyMy8jKe8chCEfr8NbeqGghy7oOCi4 #fake
    
    
    apn:
      feedback: false
      pushKey: /path/to/your/apnpushcertificates/apns-dev-key-noenc.pem #fake
      voipKey: /path/to/your/apnpushcertificates/apns-dev-key-noenc.pem #fake
      voipCertificate: /path/to/your/apnpushcertificates/apns-dev-cert.pem #fake
      pushCertificate: /path/to/your/apnpushcertificates/apns-dev-cert.pem #fake
    
    server:
        applicationConnectors:
        - type: http
          port: 9090
        adminConnectors:
        - type: http
          port: 9091
        gzip:
            enabled: true
    
    logging:
      level: INFO
      appenders:
        - type: file
          currentLogFilename: /tmp/pushserver.log
          archivedLogFilenamePattern: /tmp/pushserver-%d.log.gz
          archivedFileCount: 5
        - type: console


###The Database
Once you have PostresSQL Up&Running with username, password, messagesdb and accountsdb (two db can be the same one) **and** these parameters are well coded inside TSS' yml  file as seen above as jdbc string , you can now create data structures needed by the application.
Note that you'll not fine any sql script into the projects' sources because TSS is based on Dropwizard[^3] and Dropwizard[^3] uses Liquibase[^4] as «*open source database-independent library for tracking, managing and applying database schema changes*» so DB infos are into xml files under "src/main/resources" folder.
For executing those scripts use:

    java -jar jar/TextSecureServer-*.jar accountdb migrate config/textsecure.yml

and

    java -jar jar/TextSecureServer-*.jar messagedb migrate config/textsecure.yml

**ATTENTION** use  my[^5] repository scripts because those hosted into OWS  *now* doesn't  work because of a duplicate table definition.

###APN activation
*Do this step if you plan to use iOS clients*

Obtain  a certificate from Apple for APN service (probably a \*.cer), import in a Key Tool and export it splitted into two p12 file (apns-dev-cert.p12 and apns-dev-key.p12 for example); one for the certificate and the other for the related key. Now go to terminal and execute:

    openssl pkcs12 -clcerts -nokeys -out apns-dev-cert.pem -in apns-dev-cert.p12

**Do NOT digit a password!**
and:

    openssl pkcs12 -nocerts -out apns-dev-key.pem -in apns-dev-key.p12

At the *first* time **Do NOT digit a password!** at the *second* one **type a password** should doesn't matter which.
Now:

    openssl rsa -in apns-dev-key.pem -out apns-dev-key-noenc.pem

copy  apns-dev-cert.pem and apns-dev-key-noenc.pem somewhere into the File System reachable from PushServer and use them into PushServer's yml files

##(Temporary) APN deactivation
*useful part if you plan to test Android only as first*

Remove @NotEmpty annotation above "private ApnConfiguration apn;" field definition   into org.whispersystems.pushserver.PushServerConfiguration class, so into yml PushServers's file the APN related params become optional.

Into org.whispersystems.pushserver.PusServer's "run" method comment every row with apnSender var and pass null in the jersey Controller registration of the  PushController:

    environment.jersey().register(new PushController(null, gcmSender));

intead of

    environment.jersey().register(new PushController(apnSender, gcmSender));


###Let's start the servers

Use:

    java -jar jar/TextSecureServer-\*.jar server migrate config/textsecure.yml

and, in a separated console (servers **are not** daemonized so you have to open another console or send them in background using (in \*nix systems) "\&"):

    java -jar jar/TextSecureServer-\*.jar server config/textsecure.yml

It doesn't matter the order. Take note about server's IP


##The Android part
This paper starts with Android because it's simpler than iOS and it is simpler because Android doesn't require SSL (HTTPS) for work. **Only** for debug purposes, as you can see in TSS configuration file snippet, the SSL related parameters are commented. Additional informations will be available later in this paper because in a production environment SSL matter!!!

Start with cloning my Signal-Android repo:

    git clone --depth 1 https://github.com/lucaconte/Signal-Android.git

Now into build.gradle find "TEXTSECURE_URL" and change it with your server's IP. Mind the protocol!!!

Locate org.thoughtcrime.securesms.jobs.GcmRefreshJob and change REGISTRATION_ID static field value with senderId specified into TSS confi file. In our case: "888789650296"

It's not mandatory but if you wish to use Geo Localization probably you need to change AndroidManifest.xml setting these lines:

     <meta-data
            android:name="com.google.android.geo.API_KEY"
            android:value="AIudSyGzY1S2RcjRQh6IrUhiW5FWZEj36uRMnQs"/>

with a KEY value obtained int he Google Cloud Console. If requested, in API activation, you have to specify the package indicated at the beginning of AndroidManifest.xml, in our case: "org.thoughtcrime.securesms" 

That's all! 
Now you shold be able to play with your own Signal installation.

##SSL protection
"Play" is funny but not serious, in a real situation SSL is a MUST. So let's start with generating certificates. Create  "gencerts" bash script with the following content:

    #/bin/bash
    #This script creates root CA and server certificates to be used by the client and the server.
    # rootCA.crt needs to be copied to the client to replace the system-wide root CA set
    # example.keystore needs to be referenced by keyStorePath in the server's config file
    #
    #TO EXECUTE FROM THE OUTSIDE WITH:
    #ALTNAME=DNS:signal.foo.org ./gencerts
    #
    #IF YOU HAVE A DOMAIN OR IP if U are in a lan
    #ALTNAME=IP:10.1.4.218 ./gencerts
    #
    #
    # Create private key for root CA certificate
    openssl genrsa -out rootCA.key 4096
    
    # Create a self-signed root CA certificate
    openssl req -x509 -new -nodes  -days 3650 -out rootCA.crt -key rootCA.key
    
    # Create server certificate key
    openssl genrsa -out whisper.key 4096
    
    # Create Certificate Signing Request
    openssl req -new -key whisper.key -out whisper.csr
    
    # Sign the certificate with the root CA
    
    openssl x509 -req -in whisper.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -days 365 -out whisper.crt -extensions extensions -extfile <(cat <<-EOF
    [ extensions ]
    basicConstraints=CA:FALSE
    subjectKeyIdentifier=hash
    authorityKeyIdentifier=keyid,issuer
    subjectAltName=$ALTNAME
    EOF
    )
    
    # Export to host key and certificate to PKCS12 format which is recognized by Java keytool
    openssl pkcs12 -export -password pass:example -in whisper.crt -inkey whisper.key -out keystore.p12 -name example -CAfile rootCA.crt
    
    # Import the host key and certificate to Java keystore format, so it can be used by dropwizard
    keytool -importkeystore -srcstoretype PKCS12 -srckeystore keystore.p12 -srcstorepass example -destkeystore example.keystore -deststorepass example
    
    #whisper.store bust be placed into  Android client
    keytool -importcert -v -trustcacerts -file whisper.crt -alias IntermediateCA -keystore whisper.store -provider org.bouncycastle.jce.provider.BouncyCastleProvider -providerpath ~/data/java/Signal_prj/bcprov-jdk15on-154.jar -storetype BKS -storepass whisper
    
    #for iOS client you have to convert  whisper.crt in  DER format and install whisper.cer in Signal-iOS
    openssl x509 -in whisper.crt -out whisper.cer -outform DER

Execute this script as explained in the first lines of it. You can find the original version of this script here (Thanks to the author for sharing it):

   https://github.com/janimo/textsecure-docker
   
Again, reading the comments, should be clear that:

- "example.keystore" is the same you have specified  into TSS ycm file (uncomment the commented rows and chance protocol fotm http to https)
- "whisper.store" must be installed into Signal-Android into "res/row"
- "whisper.cer" must be installed into Signal-iOS

If you change the password ("whisper") in whisper.store generation command  you have to use the same used into the org.thoughtcrime.securesms.push.TextSecurePushTrustStore.java
If you change the password ("example") in example.keystore generation command  you have to use the same used into the TSS yml config file

##The iOS part
*Remember if didn't done before to perform APN activation (see [APN activation](#apn-activation)) and related following step*

    git clone --depth 1 https://github.com/lucaconte/Signal-iOS.git

Go to TSConstants.h and change server's urls with your own:

    #define textSecureWebSocketAPI @"wss://your_server:8080/v1/websocket/"
    #define textSecureServerURL @"https://your_server:8080/"

Edit Release.m by changing this:

    andServerPort:31337
    andMasterServerHostName:@"master.whispersystems.org"
    andDefaultRelayName:@"relay"
    andRelayServerHostNameSuffix:@"whispersystems.org"

With:

    andServerPort:8080
    //andServerPort:31337
    andMasterServerHostName:@"yourserver.foo"
    //andMasterServerHostName:@"master.whispersystems.org"
    andDefaultRelayName:@"relay"
    andRelayServerHostNameSuffix:@"yourserver.foo"
    //andRelayServerHostNameSuffix:@"whispersystems.org"

Remember to substitute "whisper.cer" (see SSL part in previous steps) with your own.

Into VertionMigrations.m change this:

    // VOIP Push might need to be enabled because 1) user ran old version 2) Update to compatible iOS version
    if (VOIPRegistration && [TSAccountManager isRegistered]) {
        [self nonBlockingPushRegistration];
    }
    
in this (added a PushRegistration call):

    // VOIP Push might need to be enabled because 1) user ran old version 2) Update to compatible iOS version
    if (VOIPRegistration && [TSAccountManager isRegistered]) {
        [self nonBlockingPushRegistration];
    }
    
    [self nonBlockingPushRegistration];

If you plan to perform a pods update go into SDatabaseSecondaryIndexes.m and change:

    //[AC 21/03/2016] Modificato perché le librerie sono state aggiornate e il metodo era deprecated
    //    YapDatabaseSecondaryIndexWithObjectBlock block =
    //    ^(NSMutableDictionary *dict, NSString *collection, NSString *key, id object) {
    //        
    //        if ([object isKindOfClass:[TSInteraction class]]) {
    //            TSInteraction *interaction = (TSInteraction *)object;
    //            
    //            [dict setObject:@(interaction.timestamp) forKey:TSTimeStampSQLiteIndex];
    //        }
    //    };

    YapDatabaseSecondaryIndexWithObjectBlock block =
        ^(YapDatabaseReadTransaction *transaction, NSMutableDictionary *dict, NSString *collection, NSString *key, id object) {

          if ([object isKindOfClass:[TSInteraction class]]) {
              TSInteraction *interaction = (TSInteraction *)object;

              [dict setObject:@(interaction.timestamp) forKey:TSTimeStampSQLiteIndex];
          }
        };
    YapDatabaseSecondaryIndexHandler *handler = [YapDatabaseSecondaryIndexHandler withObjectBlock:block];

Now you can update the pods

**That's All!!**

Many thanks to anyone that helped me in reaching the end

All feedbacks are welcome


---
<a name="one"></a>1: http://support.whispersystems.org/hc/en-us/articles/212477768 "OWS security"

<a name="two"></a>2: Originally https://github.com/WhisperSystems/TextSecure-Server/issues/44 but it has been removed 

<a name="three"></a>3: http://dropwizard.io

<a name="four"></a>4: http://www.liquibase.org/

<a name="five"></a>5: https://github.com/lucaconte/TextSecure-Server/tree/master/src/main/resources
