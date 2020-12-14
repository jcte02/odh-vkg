FROM keycloak/keycloak-gatekeeper:7.0.0

COPY infrastructure/docker/gatekeeper/configuration.yml /etc/gatekeeper/configuration.yml

ENTRYPOINT [ "/opt/keycloak-gatekeeper", "--config", "/etc/gatekeeper/configuration.yml" ]
