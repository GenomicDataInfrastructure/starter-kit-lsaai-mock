![](logo_small.png)

# GDI-LSAAI-MOCK

Sample docker-compose to bring up and run a mock of the LS AAI. Contains a predefined user and sample client. Further configuration is expected to add clients and users for the testing.

## Running

Use the provided docker-compose to run an example deployment. Update the configurations of clients, users and GA4GH Visas if needed in the mapped configuration directory and with the help of the `Configuration` section below.

### AAI (login) mock

When you run the sample docker-compose, you will have AAI mock running on port `8080` with two paths exposed:

* `localhost:8080/oidc/` - the mock of the AAI which provides login and OpenID Connect provider functionality
* `localhost:8080/ga4gh-broker/` - the mock of the LS AAI GA4GH Passport broker

Visit the `localhost:8080/oidc/.well-known/openid-configuration` endpoint to learn about the endpoints of the OpenID Provider.

#### Bypassing the user selection

To bypass the user selection, while sending out the initial request to the authorization endpoint, you can provide an additional parameter `username_hint`, with a value matching to one of the user identifiers from the configured userinfo responses. This will trigger pre-selection of the user and can be used e.g. in the automated testing scenario to completely bypass the user "authentication" part, resulting in streamlining the login simulation.

### Broker

You do not need to interact with the broker at all in the usual use cases. If for some reason you have to, the mapping of the endpoints is usually:

* `/ga4gh-broker/jwks` - endpoint exposing the JWKS used for visa signature verification
* `/ga4gh-broker/1.0/{user_id}` - fetch visas using the Standard v1.0 procedure - generates list of visas (replace user_id with the userId from configuration file)
* `/ga4gh-broker/1.2/` - fetch visas using the Standard v1.2 procedure - generates GA4GH Passport token (has to be called with Bearer token validated against the local AAI mockup)

## Volumes

The sample docker-compose contains a mapping of the local directory into the containers.

- `/etc/lsaai-mock/` - directory with all configuration

## Databases

This docker expects a MySQL Database to be available for both applications running in it.
When creating databases, use the provided *.sql scripts to initialize tables.

## Configuration

### AAI Mock

The whole configuration needs to be located under `/etc/lsaai-mock/` directory. The structure should look like:

```
  +- application.properties
  +- keys.jwks
  +- clients
     +- client1.yaml
     +- client2.yaml
     +- ...
  +- userinfos
     +- user1.yaml
     +- user2.yaml
     +- ...
```

#### application.properties

This is the main configuration file of the AAI OpenID Provider mock.
```
### URL where the application runs
main.oidc.issuer.url=http://localhost:8080/oidc/
### base path for web (e.g. issuer without trailing slash)
web.baseURL=https://localhost:8080/oidc

# configuration of the GA4GH broker location and authentication
ga4gh.broker.url=http://localhost:8081/ga4gh-broker/
ga4gh.broker.username=user
ga4gh.broker.password=pass

# db
jdbc.url=jdbc:mysql://localhost:3306/lsaai-mock
jdbc.user=lsaai-mock
jdbc.password=password
```

You can generate your own keys or use the ones provided in this repository (file `keys.jwks`).

#### userinfos

To configure user information and data used for ID token and userinfo responses, create one file per user in the `userinfors` directory.
Sample user file:

```
---
# How the user will be named on the login page
web-display-name: "Sample user"

### Information released in the userinfo response / ID token
### claims commented out are not released by the LS AAI

# scope openid
sub: user@lifescience-ri.euň

# scope profile
name: "John Doe"
preferred-username: "jd123"
given-name: "John"
family-name: "Doe"
middle-name: "Franklin"
nickname: "johnny"
#profile: 
#picture:
#website:
#gender:
#zoneinfo:
locale: "EN"
#updated-at: 
#birthdate:

# scope email
email: "jd@edu.com"
email-verified: True

# scope phone
#phone-number:
#phone-number-verified:

# scope address
#address:

# scope country
country: "uk"

# scope eduperson-entitlement
eduperson-entitlement: ["urn:geant:lifescience-ri:group:sampleUsers#lifescience-ri.eu"]

# scope voperson-external-affiliation
voperson-external-affiliation: ["unknown@orcid.org", "affiliate@edu2.edu"]

# scope eduperson-orcid
eduperson-orcid: "my-orcid@orcid.org"

# scope schac-home-organization
schac-home-organization: ["edu.com", "edu2.edu"]

# scope eduperson-scoped-affiliation
eduperson-scoped-affiliation: ["affiliate@lifescience-ri.eu"]

# scope voperson-current-external-affiliation
voperson-current-external-affiliation : ["affiliate@edu2.edu"]

# scope authenticating-entity
authenticating-entity:
```

Scope `ga4gh_passport_v1` does not need to be configured as it is directly read from the broker instance.


#### clients

To configure clients able to use the OpenID Provider, create one file per client in the `clients` directory.
Sample client file:

```
# Webpages display the name of the client
client-name: "Sample client"

# Unique identifier of the client
client-id: "app-123"

# Secret value used by the client as a part of the authentication in communication with the AAI
client-secret: "secret_value"

# Redirect URIs where the response from the OP will be handled
redirect-uris: ["http://localhost:9009/oidc-redirect"]

# How the client authenticates towards the token endpoint - one of 'client_secret_post', 'client_secret_basic', 'none'. In most cases, "client_secret_basic" is the correct value.
token-endpoint-auth-method: "client_secret_basic"

# List of the scopes the client can request
scope: ["openid", "profile", "email", "ga4gh_passport_v1"]

# Supported grant types - subset of 'authroization_code', 'implicit', 'urn:ietf:params:oauth:grant-type:token-exchange'. In most cases, "authorization_code" is the correct value.
grant-types: ["authorization_code"]

# Where the user can be redirected after logging out
post-logout-redirect-uris: ["http://localhost:9009/post-logout"]

# How many seconds is the access token valid
# access-token-validity-seconds: 3600

# How many seconds is the refresh token valid
# refresh-token-validity-seconds: 7200

#id-token-validity-seconds:
# Only needed if you want to use the device flow
# device-code-validity-seconds: 123

# Only configure this if you want to enable PKCE flow - set to 'plain' or 'S256'. In such a case the token-endpoint-auth-method should be set to 'none'.
# code-challenge-method: "S256"

# Resource IDs if you plan to make use of Resource Indicators (RFC 8707)
# Configures values of allowed "resource" params. When performing authN request and resource indicators are passed, OIDC will check if all values are registered either
# directly as client-id(s) or as resource-id(s) in some configured client
# resource-ids: ['app-123', 'broker-cid', 'urn:gdi:beacon']
```

This directory has to include configuration for the broker client if you want to use `GA4GH Passport and Visas standard v1.2`:

```
---
client-name: "GA4GH Broker Client"
client-id: "broker"
client-secret: "broker-secret"
redirect-uris: []
token-endpoint-auth-method: "client_secret_basic"
scope: ["openid", "ga4gh_passport_v1"]
grant-types: []
```

### GA4GH Passport broker

The broker collects the GA4GH Visas from the configured sources and releases them via the AAI mock to the requesting clients.

The whole configuration needs to be located under `/etc/lsaai-mock/ga4gh-broker/` directory. The structure should look like:

```
  +- application.yaml
  +- users
     +- user1.yaml
     +- user2.yaml
     +- ...
```

#### application.yaml

```
---

spring:
  main:
    allow-bean-definition-overriding: true
  mvc:
    pathmatch:
      matching-strategy: ant_path_matcher

logging:
  level:
    root: info
    cz.muni.ics.ga4gh: debug
    org.springframework.web: debug

mock-broker:
  enabled: true
  # Issuer from the AAI mock configuration file
  issuer: http://localhost:8080/oidc/
  # Where our JWKS will be available - fill in the host part, e.g. http://loclahost:8081
  jku: [host]/ga4gh-broker/jwk
  # Location of the physical JWKS - point to the same location as stored in the AAI Mock
  path-to-jwk-file: /etc/lsaai-mock/keys.jwk
  # Where user configurations are located
  visa-configs-path: /etc/lsaai-mock/ga4gh-broker/users/
  # External passport repositories - we will fetch visas via API calls from these locations
  passport-repositories:
      # For readability purposes
    - name: Sample-REMS
      # Location to fetch the visas - {user_id} part will be replaced with the identifier of the user - contents of the claim SUB from AAI mock
      url: http://rems.local/api/permissions/{user_id}
      # Location of JWKS
      jwks: http://rems.local/api/jwk
      # Headers for authentication purposes
      headers:
        - header: x-rems-api-key
          value: sample-key
        - header: x-rems-user-id
          value: sample-user

# Enables functionality to store visas by calling the API - e.g. REMS can write proactively Visas into the broker to save the network communication
visa-writers:
  enabled: True
  # Database connection
  db:
    url: jdbc:mysql://localhost:3307/ga4gh_broker
    username: ga4gh_broker
    password: password
    driver: "com.mysql.cj.jdbc.Driver"
  # Repositories that are able to write
  repositories:
    - name: Writer-REMS
      # Lcoation of JWKS
      jwks: http://writer-rems.local/jwk
      auth:
        # can be one of
        api-key:
          api-key-header: x-visa-writer
          api-key-value: sample-key
        basic-auth:
          username: visa-writer
          password: password

# Endpoint to fetch visas using Passport and Visa standard v1.2 - token exchange
passport-v1-2:
  auth:
    # Client ID from the AAI mock configuration for broker
    introspection-client-id: broker
    # Client secret from the AAI mock configuration for broker
    introspection-client-secret: broker-secret
    # Introspection endpoint of the AAI mock
    introspection-uri: http://localhost:8080/oidc/introspect

# Endpoint to fetch visas using Passport and Visa standard v1.0 - userinfo method
passport-v1-0:
  # Credentials that will be used by AAI mock to contact this broker - see application.properties in AAI mock
  auth:
    - username: user
      password: pass
```

#### users

The visas issued directly by the broker are configured per user via a file per user stored in the `users` (or according to your configuration) directory.
Sample user file:

```
---
# Identifier of the user - has to match the value of the 'sub' claim in AAI
userId: user@lifescience-ri.eu
visas:
    # Who has issued the visa
  - source: https://organization.edu
    # Number of seconds from epoch when the visa will expire.
    expires: 12345679
    # Number of seconds from epoch when the visa has been asserted.
    asserted:
    # Value of the visa
    value: test
    # Type of the visa - one of 'AffiliationAndRole', 'AcceptedTermsAndPolicies', 'ControlledAccessGrants', 'LinkedIdentities', 'ResearcherStatus' 
    type: AffiliationAndRole
    # Who has asserted the Visa
    by: "system"
    # Condiition - usually empty
    #conditions:
```
