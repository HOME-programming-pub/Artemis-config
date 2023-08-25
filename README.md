# Artemis-config
Setup of the Artemis-Plattform based on Gitlab and Docker.

Command for starting the application using Docker-Compose (order of yml-files matters):
```
docker compose -f gitlab.yml -f mysql.yml -f artemis.yml up -d
```

Command for stopping the application:
```
docker compose -f gitlab.yml -f mysql.yml -f artemis.yml down
```

## Local Test Setup

### GitLab

If started for the first time, a Gitlab instance runner needs to be created in the Gitlab UI (see [GitLab Documentation](https://docs.gitlab.com/ee/ci/runners/runners_scope.html#shared-runners) for more details). If the GitLab container is up and fully started, log in with the root account and go to the [runner configuration site](https://localhost/admin/runners/new). The UI spills out a registration token, which must be inserted into the runner's Docker container ``config.toml`` configuration file, as a value of the ``token`` property. This allows the actual runner process to authenticate with the Gitlab server. 

Furthermore, a Gitlab personal access token for all scopes is required and needs to be placed into the local Artemis configuration file ``prod-application-local.env`` as a value of the ``ARTEMIS_VERSIONCONTROL_TOKEN`` variable. Additionally, it might be set for the ``ARTEMIS_VERSIONCONTROL_HEALTHAPITOKEN`` variable. However, in this case, a token with readonly scopes should be sufficient (would need to be generated separately). The personal access token must be created using an account with sufficient permissions (e.g., root) in the [user's preferences](https://localhost/-/profile/personal_access_tokens).

### User-generated Certificates (SSL)

A local test setup should come close to a production setup and hence should also have SSL enabled. To mimic a production setup has several steps involved to generate a chain of self-signed SSL keys and certificates that can be used encrypt the communication between the services managed by Docker. These services have internal names and addresses within the Docker network, which is mostly hidden from the actual host, except the ports that are mapped via port mappings in the yml-files.

#### Generating PEM Files

While this repo already contains a set of locally-trusted certificate, you may want to generate some on your own using your own domain names. A fairly simple tool to do so is mkcert (https://github.com/FiloSottile/mkcert). Our project https://github.com/HOME-programming-pub/mkcert-docker exemplifies how to use mkcert to generate the set of files required by clients and serves, including a local root certificate authority. Using that script spilled out the files (already copied to this project)
* [docker/gitlab/certs/artemis.hs-merseburg.de+8-key.pem](docker/gitlab/certs/artemis.hs-merseburg.de+8-key.pem) (the private key used by the server),
* [docker/gitlab/certs/artemis.hs-merseburg.de+8.pem](docker/gitlab/certs/artemis.hs-merseburg.de+8.pem) (the certificate used by the browser) and
* [docker/gitlab-runner/certs/rootCA.pem](docker/gitlab-runner/certs/rootCA.pem) (the CA root certificate).

#### Browser and GitLab

To enable a trusted experience without complaints and error messages, the root certificate needs to be placed into the CA store of the browser's or the host's CA store, the GitLab-Runner(s) and Artemis. The private key and the certificate are needed by GitLab (and nginx respectively) and Artemis. For the client browser, GitLab and GitLab runner, the generated pem-files can be used as they are. They just need to be mapped to the Docker-volumes and locations expected by the software. The browser installation is vendor-specific. [This link](https://support.mozilla.org/de/kb/zertifizierungsstellen-firefox-einrichten) describes how it works in Firefox.

#### Artemis (Spring and Java)
For Artemis, some more complicated steps are involved. As Artemis is based on Java and Spring, the SSL-certifcates need some additional massage. [Spring requires an SSL bundle](https://spring.io/blog/2023/06/07/securing-spring-boot-applications-with-ssl) which stores the certificates in PKCS12 format or as a [Java keystore](https://en.wikipedia.org/wiki/Java_KeyStore). A combined PKS-Keypair for Java's keystore kann be generated with [OpenSSL](https://www.openssl.org/) from given certificates using the following command: 
```
openssl pkcs12 -export -in artemis.hs-merseburg.de+8.pem -inkey artemis.hs-merseburg.de+8-key.pem -out all_converted.pfx -certfile rootCA.pem
```
The generated file (``all_converted.pfx``) can be used by Spring as a keystore file (ssl bundle). 

Additionally, Artemis communicates via Java's SSL API with Gitlab. This requires to add the generated ``rootCA.pem`` to the JVM instance that runs in the Artemis Docker container. This can be done by updating an existing Java CA keystore (e.g., on the host machine): 
```
sudo keytool -importcert -alias mycert-local -file rootCA.pem -keystore %JAVA_HOME%/lib/security/cacerts
``` 
Alternatively, keystores kann be edited using a graphical tool like [keystore explorer](http://keystore-explorer.org/index.html).
The updated cacerts-file must then be mounted in the JVM in the Artemis Docker container (location ``%JAVA_HOME%/lib/security/cacerts``).

#### Mysql
MySQL can be configured to enforce encrypted communication using the plain PEM files in the user configuration file [my.cnf](docker/mysql/my.cnf) by setting ``require_secure_transport = 1``. It will then search certificates in the data directory, which, by default, is ``/var/lib/mysql``. To make MySQL use our generated PEM files, they must be mapped to that directory and configured in [my.cnf](docker/mysql/my.cnf) as follows:
```
ssl_ca = rootCA.pem
ssl_cert = artemis.hs-merseburg.de+8.pem
ssl_key = artemis.hs-merseburg.de+8-key.pem
```
Note that the MySQL container may fail when you run it for the first time, issueing the message ``--initialize specified but the data directory has files in it. Aborting.`. This indicates that it found files in the data directory, which is expected to be empty on the first run. In that case, it must be ensured that no certificates are mapped to the data directory when it is initialized. Just comment out the certifcate mappings in the compose file for the first run.

### Setup Procedure: Summary
The following list summarizes the involved setup steps to get Artemis up an running with GitLab.

1. Clone this repo to your machine including the Artemis submodule
1. Generate (local setup)/ obtain (production setup) SSL Certifcates
1. Convert and configure SSL Certificates (PEM files, PFX files, Java keystore)
1. Start GitLab and copy the personal access token of root user, and create an instance runner and copy the runner token
1. Add the GitLab tokens to the configuration files
1. Restart GitLab and also start GitLab runner; check if the runner registers with GitLab and listens for jobs
1. Start MySQL for the first time, without the PEM files (initialization)
1. Restart MySQL with the PEM files; check if it remains online
1. Stop and remove all running containters
1. Start all containers with ``docker compose -f gitlab.yml -f mysql.yml -f artemis.yml up -d``
1. Check if the artemis_admin user exists in Artemis and GitLab (note: "Password must not contain commonly used combinations of [English] words and letters")

Don't forget to change user and passwords. The defaults are good for testing only.