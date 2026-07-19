# FAQ

### Basic (username password) authentication

[basic-authentication.md](../configuration/authentication/for-the-ui/basic-authentication.md "mention")

### Role-based access control

[rbac-role-based-access-control](../configuration/rbac-role-based-access-control/ "mention")

### OAuth 2

[oauth2.md](../configuration/authentication/for-the-ui/oauth2.md "mention")

### LDAP

See [this](https://github.com/kafbat/kafka-ui/blob/main/documentation/compose/auth-ldap.yaml#L29) example.

### Active Directory (LDAP)

See [this](https://github.com/kafbat/kafka-ui/blob/main/documentation/compose/auth-ldap.yaml#L29) example.

### SAML

Planned, see [#478](https://github.com/kafbat/kafka-ui/issues/478)

### Message filtering / smart filters syntax

[filtering.md](filtering.md "mention")

### Can I use the app as API?

Sure! Swagger declaration is located [here](https://github.com/kafbat/kafka-ui/blob/main/contract/src/main/resources/swagger/kafbat-ui-api.yaml).

### My OIDC / OAuth provider uses self-signed certificates, how do I add them to the truststore?

```yaml
server:
  ssl:
    trust-store: classpath:keycloak-truststore.jks
    trust-store-password: changeit
```
