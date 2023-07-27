# Artemis-config
Artemis setup of Hochschule Merseburg based on Gitlab.

Command for starting the application using Docker-Compose:
``docker compose -f gitlab.yml -f mysql.yml -f artemis.yml up -d``

If started for the first time, a Gitlab instance runner needs to be created in the Gitlab UI. The UI spills out a registration token, which must be inserted into the runner's ``config.toml`` configuration file, as a value of the ``token`` property.

Furthermore, a Gitlab personal access token for all scopes is required and needs to be placed into the local Artemis configuration file ``prod-application-local.env`` as a value of the ``ARTEMIS_VERSIONCONTROL_TOKEN`` variable. Additionally, it might be set for the ``ARTEMIS_VERSIONCONTROL_HEALTHAPITOKEN`` variable. However, in this case, a token with readonly scopes should be sufficient (would need to be generated separately).
