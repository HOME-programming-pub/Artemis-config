# Artemis-config
Setup of the Artemis-Plattform based on Gitlab and Docker.

Command for starting the application using Docker-Compose (order of yml-files matters):
```docker compose -f gitlab.yml -f mysql.yml -f artemis.yml up -d```

Command for stopping the application:
```docker compose -f gitlab.yml -f mysql.yml -f artemis.yml down```

## Local Test Setup

### GitLab

If started for the first time, a Gitlab instance runner needs to be created in the Gitlab UI. The UI spills out a registration token, which must be inserted into the runner's ``config.toml`` configuration file, as a value of the ``token`` property.

Furthermore, a Gitlab personal access token for all scopes is required and needs to be placed into the local Artemis configuration file ``prod-application-local.env`` as a value of the ``ARTEMIS_VERSIONCONTROL_TOKEN`` variable. Additionally, it might be set for the ``ARTEMIS_VERSIONCONTROL_HEALTHAPITOKEN`` variable. However, in this case, a token with readonly scopes should be sufficient (would need to be generated separately).

### User-generated Certificates (SSL)

A local test setup should come close to a production setup and hence should also have SSL enabled. To mimic a production setup has several steps involved to generate a chain of self-signed SSL keys and certificates that can be used encrypt the communication between the services managed by Docker. These services have internal names and addresses within the Docker network, which is mostly hidden from the actual host, except the ports that are mapped via port mappings in the yml-files.

While this repo already contains a set of locally-trusted certificate, you may want to generate some on your own using your own domain names. A fairly simple tool to do so is mkcert (https://github.com/FiloSottile/mkcert). Our project https://github.com/HOME-programming-pub/mkcert-docker exemplifies how to use mkcert to generate the set of files required by clients and serves, including a local root certificate authority. Using that script spills out the files [docker/gitlab/certs/artemis.hs-merseburg.de+8-key.pem](docker/gitlab/certs/artemis.hs-merseburg.de+8-key.pem) (the private key used by the server), [docker/gitlab/certs/artemis.hs-merseburg.de+8.pem](docker/gitlab/certs/artemis.hs-merseburg.de+8.pem) (the certificate used by the browser) and [docker/gitlab-runner/certs/rootCA.pem](docker/gitlab-runner/certs/rootCA.pem) (the CA root certificate).

To enable a trusted experience without complaints and error messages, the root certificate needs to be placed into the CA store of the browser's or the host's CA store, the GitLab-Runner(s) and Artemis. The private key and the certificate are needed by GitLab (and nginx respectively) and Artemis. For the browser, GitLab and GitLab runner, the generated pem-files can be used as is. They just need to be mapped to the Docker-volumes and locations expected by the software.  [TODO: MySQL] 

For Artemis, this involves some more complicated steps. As Artemis is based on Java and Spring, the SSL-certifcates need some additional massage. Spring requires a Java keystore which stores the certificates in PKCS12 format. A combined PKS-Keypair for Java's keystore kann be generated from the given certificates using the following command: ``openssl pkcs12 -export -in artemis.hs-merseburg.de+8.pem -inkey artemis.hs-merseburg.de+8-key.pem -out all_converted.pfx -certfile rootCA.pem``. The generated file (all_converted.pfx) can be used by Spring as a keystore file. 

Additionally, Artemis communicates via Java's SSL API with Gitlab. This requires to add the generated rootCA to the running JVM instance. This can be done by updating an existing Java CA-store: ``sudo keytool -importcert -alias mycert-local -file rootCA.pe
m -keystore %JAVA_HOME%/lib/security/cacerts`` and mount the updated cacerts-file to the JVM in the Artemis Docker container.


