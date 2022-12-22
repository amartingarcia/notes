---
title: Keycloak - Sonarqube
date: 20221212
author: Adrián Martín García
---

# Sonarqube
In this page we define the configuration for a correct integration between Sonarqube and Keycloak.

## Requirements
* [Sonarqube](https://github.com/SonarSource/helm-chart-sonarqube/tree/master/charts/sonarqube)
* [Plugin](https://github.com/vaulttec/sonar-auth-oidc)

## Configuration
First we will need to configure Keycloak. We will assume that we have a new Realm called **Factory**.

### Keycloak
#### Clients
Create **sonarqube** user.
![notes](../images/security/keycloak/sonarqube_clients.png)

#### Client Scope
Declare **Groups** scope in client.
![notes](../images/security/keycloak/sonarqube_client_scope.png)

#### Groups
Create groups:

* **sonar-administrators**
![notes](../images/security/keycloak/sonarqube_groups.png)

#### Users
Join user to a **sonar-administrators** group.
![notes](../images/security/keycloak/sonarqube_users.png)

#### Mappers
Create a **Mapper** in **Identity Provider**.
![notes](../images/security/keycloak/sonarqube_mappers_01.png)
![notes](../images/security/keycloak/sonarqube_mappers_02.png)

### Sonarqube
At this point we define the necessary configuration in Sonarqube to be able to perform the integration with Keycloak.

#### Configuration
The YAML file for the Helm Chart is:
```yaml
ingress:
  enabled: true
  hosts:
    - name: sonarqube.<your-domain>
      path: /
  annotations:
    external-dns.alpha.kubernetes.io/hostname: sonarqube.<your-domain>
    nginx.ingress.kubernetes.io/proxy-body-size: "8m"
  ingressClassName: nginx

prometheusExporter:
  enabled: false

jdbcOverwrite:  
  enable: false

sonarProperties:
  sonar.core.serverBaseURL: "https://sonarqube.<your-domain>"
  sonar.auth.oidc.enabled: true
  sonar.auth.oidc.issuerUri: "https://keycloak.<your-domain>/auth/realms/factory"
  sonar.auth.oidc.clientId.secured: "sonarqube"
  sonar.auth.oidc.scopes: "openid email profile groups"
  sonar.auth.oidc.groupsSync: true

plugins:
  install:
    - "https://github.com/vaulttec/sonar-auth-oidc/releases/download/v2.1.1/sonar-auth-oidc-plugin-2.1.1.jar"

postgresql:
  enabled: true

account:
    adminPassword: <username>
    currentAdminPassword: <userpassword>
```

#### Sonarqube - Groups
`sonar-administrators` and `sonar-users` groups already exists.
![notes](../images/security/keycloak/sonarqube_role_groups.png)

## Login
![notes](../images/security/keycloak/sonarqube_login_01.png)
![notes](../images/security/keycloak/sonarqube_login_02.png)
