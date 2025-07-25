---
navigation_title: JWT
mapped_pages:
  - https://www.elastic.co/guide/en/cloud/current/ec-securing-clusters-JWT.html
  - https://www.elastic.co/guide/en/cloud-enterprise/current/ece-securing-clusters-JWT.html
  - https://www.elastic.co/guide/en/cloud-heroku/current/ech-securing-clusters-JWT.html
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/jwt-auth-realm.html
applies_to:
  deployment:
    self:
    ess:
    ece:
    eck:
products:
  - id: cloud-hosted
  - id: cloud-enterprise
  - id: elasticsearch
---

# JWT authentication [jwt-auth-realm]

{{es}} can be configured to trust JSON Web Tokens (JWTs) issued from an external service as bearer tokens for authentication.

When a JWT realm is used to authenticate with {{es}}, a distinction is made between the client that is connecting to {{es}}, and the user on whose behalf the request should run. The JWT authenticates the user, and a separate credential authenticates the client.

The JWT realm supports two token types, `id_token` (the default) and `access_token`:

1. `id_token`: An application authenticates and identifies a user with an authentication flow, e.g. OpenID Connect (OIDC), and then accesses {{es}} on behalf of the authenticated user using a JSON Web Token (JWT) conforming to OIDC ID Token specification. This option is available in deployments using {{stack}} 8.2+.
2. `access_token`: An application accesses {{es}} using its own identity, encoded as a JWT, e.g. The application authenticates itself to a central identity platform using an OAuth2 Client Credentials Flow and then uses the resulting JWT-based access token to connect to {{es}}. This option is available in deployments using {{stack}} 8.7+.

::::{note}
A single JWT realm can only work with a single token type. To handle both token types, you must configure at least two JWT realms. You should choose the token type carefully based on the use case because it impacts on how validations are performed.
::::

The JWT realm validates the incoming JWT based on its configured token type. JSON Web Tokens (JWT) of both types must contain the following 5 pieces of information. While ID Tokens, based on the OIDC specification, have strict rules for what claims should provide these information, access tokens allow some claims to be configurable.

**Claims**

| Information | ID Token | Access Token |
| --- | --- | --- |
| Issuer | `iss` | `iss` |
| Subject | `sub` | Defaults to `sub`, but can fall back to another claim if `sub` does not exist |
| Audiences | `aud` | Defaults to `aud`, but can fall back to another claim if `aud` does not exist |
| Issue Time | `iat` | `iat` |
| Expiration Time | `exp` | `exp` |

In addition, {{es}} also validates `nbf` and `auth_time` claims for ID Tokens if these claims are present. But these claims are ignored for access tokens.

Overall, the access token type has more relaxed validation rules and is suitable for more generic JWTs, including self-signed ones.

## ID Tokens from OIDC workflows [jwt-realm-oidc]

JWT authentication in {{es}} is derived from OIDC user workflows, where different tokens can be issued by an OIDC Provider (OP), including ID Tokens. ID Tokens from an OIDC provider are well-defined JSON Web Tokens (JWT) and should be always compatible with a JWT realm of the `id_token` token type. The subject claim of an ID token represents the end-user. This means that ID tokens will generally have many allowed subjects. Therefore, a JWT realm of `id_token` token type does *not* mandate the `allowed_subjects` (or `allowed_subject_patterns`) validation.

::::{note}
Because JWTs are obtained external to {{es}}, you can define a custom workflow instead of using the OIDC workflow. However, the JWT format must still be JSON Web Signature (JWS). The JWS header and JWS signature are validated using OIDC ID token validation rules.
::::


{{es}} supports a separate [OpenID Connect realm](/deploy-manage/users-roles/cluster-or-deployment-auth/openid-connect.md). It is preferred for any use case where {{es}} can act as an OIDC RP. The OIDC realm is the only supported way to enable OIDC authentication in {{kib}}.

::::{tip}
Users authenticating with a JWT realm can optionally impersonate another user with the [`run_as`](/deploy-manage/users-roles/cluster-or-deployment-auth/submitting-requests-on-behalf-of-other-users.md) feature. See [Applying the `run_as` privilege to JWT realm users](/deploy-manage/users-roles/cluster-or-deployment-auth/jwt.md#jwt-realm-runas).
::::


## Access tokens [jwt-realm-oauth2]

A common method to obtain access tokens is with the OAuth2 client credentials flow. A typical usage of this flow is for an application to get a credential for itself. This is the use case that the `access_token` token type is designed for. It is likely that this application also obtains ID Tokens for its end-users. To prevent end-user ID Tokens being used to authenticate with the JWT realm configured for the application, we mandate `allowed_subjects` or `allowed_subject_patterns` validation when a JWT realm has token type `access_token`.

::::{note}
Not every access token is formatted as a JSON Web Token (JWT). For it to be compatible with the JWT realm, it must at least use the JWT format and satisfies relevant requirements in the above table.
::::

## Configure {{es}} to use a JWT realm [jwt-realm-configuration]

To use JWT authentication, create the realm in the [`elasticsearch.yml`](/deploy-manage/stack-settings.md) file to configure it within the {{es}} authentication chain.

The JWT realm has a few mandatory settings, plus optional settings that are described in [JWT realm settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-jwt-settings).

::::{note}
Client authentication is enabled by default for the JWT realms. Disabling client authentication is possible, but strongly discouraged.
::::

1. Configure the realm using your preferred token type:

  :::::{tab-set}

  ::::{tab-item} ID tokens

  The following example includes the most common settings, which are not intended for every use case:

  ```yaml
  xpack.security.authc.realms.jwt.jwt1:
    order: 3
    token_type: id_token
    client_authentication.type: shared_secret
    allowed_issuer: "<example-issuer-url>/jwt/"
    allowed_audiences: [ "8fb85eba-979c-496c-8ae2-a57fde3f12d0" ]
    allowed_signature_algorithms: [RS256,HS256]
    pkc_jwkset_path: jwt/jwkset.json
    claims.principal: sub
  ```

  `order`
  :   Specifies a realm `order` of `3`, which indicates the order in which the configured realm is checked when authenticating a user. Realms are consulted in ascending order, where the realm with the lowest order value is consulted first.

  `token_type`
  :   Instructs the realm to treat and validate incoming JWTs as ID Tokens (`id_token`).

  `client_authentication.type`
  :   Specifies the client authentication type as `shared_secret`, which means that the client is authenticated using an HTTP request header that must match a pre-configured secret value. The client must provide this shared secret with every request in the `ES-Client-Authentication` header and using the `SharedSecret` scheme. The header value must be a case-sensitive match to the realm’s `client_authentication.shared_secret`.

  `allowed_issuer`
  :   Sets a verifiable identifier for your JWT issuer. This value is typically a URL, UUID, or some other case-sensitive string value.

  `allowed_audiences`
  :   Specifies a list of JWT audiences that the realm will allow. These values are typically URLs, UUIDs, or other case-sensitive string values.

  `allowed_signature_algorithms`
  :   Indicates that {{es}} should use the `RS256` or `HS256` signature algorithms to verify the signature of the JWT from the JWT issuer.

  `pkc_jwkset_path`
  :   The file name or URL to a JSON Web Key Set (JWKS) with the public key material that the JWT Realm uses for verifying token signatures. A value is considered a file name if it does not begin with `https`. The file name is resolved relative to the {{es}} configuration directory. If a URL is provided, then it must begin with `https://` (`http://` is not supported). {{es}} automatically caches the JWK set and will attempt to refresh the JWK set upon signature verification failure, as this might indicate that the JWT Provider has rotated the signing keys.

  `claims.principal`
  :   The name of the JWT claim that contains the user’s principal (username).

  ::::

  ::::{tab-item} Access tokens
  The following is an example snippet for configuring a JWT realm for handling access tokens:

  ```yaml
  xpack.security.authc.realms.jwt.jwt2:
    order: 4
    token_type: access_token
    client_authentication.type: shared_secret
    allowed_issuer: "<example-issuer-url>/jwt/"
    allowed_subjects: [ "123456-compute@admin.example.com" ]
    allowed_subject_patterns: [ "wild*@developer?.example.com", "/[a-z]+<1-10>\\@dev\\.example\\.com/"]
    allowed_audiences: [ "elasticsearch" ]
    required_claims:
      token_use: access
      version: ["1.0", "2.0"]
    allowed_signature_algorithms: [RS256,HS256]
    pkc_jwkset_path: "<example-idp-url>/.well-known/configuration"
    fallback_claims.sub: client_id
    fallback_claims.aud: scope
    claims.principal: sub
  ```

  `token_type`
  :   Instructs the realm to treat and validate incoming JWTs as access tokens (`access_token`).

  `allowed_subjects`
  :   Specifies a list of JWT subjects that the realm will allow. These values are typically URLs, UUIDs, or other case-sensitive string values.

  `allowed_subject_patterns`
  :   Analogous to `allowed_subjects` but it accepts a list of [Lucene regexp](elasticsearch://reference/query-languages/query-dsl/regexp-syntax.md) and wildcards for the allowed JWT subjects. Wildcards use the `*` and `?` special characters (which are escaped by `\`) to mean "any string" and "any single character" respectively, for example "a?\**", matches "a1*" and "ab*whatever", but not "a", "abc", or "abc*" (in Java strings `\` must itself be escaped by another `\`). [Lucene regexp](elasticsearch://reference/query-languages/query-dsl/regexp-syntax.md) must be enclosed between `/`, for example "/https?://[^/]+/?/" matches any http or https URL with no path component (matches "https://elastic.co/" but not "https://elastic.co/guide").

  At least one of the `allowed_subjects` or `allowed_subject_patterns` settings must be specified (and be non-empty) when `token_type` is `access_token`.

  When both `allowed_subjects` and `allowed_subject_patterns` settings are specified an incoming JWT’s `sub` claim is accepted if it matches any of the two lists.

  `required_claims`
  :   Specifies a list of key/value pairs for additional verifications to be performed against a JWT. The values are either a string or an array of strings.

  `fallback_claims.sub`
  :   The name of the JWT claim to extract the subject information if the `sub` claim does not exist. This setting is only available when `token_type` is `access_token`. The fallback is applied everywhere the `sub` claim is used. In the above snippet, it means the `claims.principal` will also fallback to `client_id` if `sub` does not exist.

  `fallback_claims.aud`
  :   The name of the JWT claim to extract the audiences information if the `aud` claim does not exist. This setting is only available when `token_type` is `access_token`. The fallback is applied everywhere the `aud` claim is used.
  ::::
  :::::

2. Add secure settings [to the {{es}} keystore](/deploy-manage/security/secure-settings.md):

   * The `shared_secret` value for `client_authentication.type`

      (`xpack.security.authc.realms.jwt.jwt1.client_authentication.shared_secret1`)
   * The HMAC keys for `allowed_signature_algorithms`

      (`xpack.security.authc.realms.jwt.jwt1.hmac_jwkset`)

      This setting can be a path to a JWKS, which is a resource for a set of JSON-encoded secret keys. The file can be removed after you load the contents into the {{es}} keystore.


  :::{note}
  Using the JWKS is preferred. However, you can add an HMAC key in string format using `xpack.security.authc.realms.jwt.jwt1.hmac_key`. This format is compatible with HMAC UTF-8 keys, but only supports a single key with no attributes. You can only use one HMAC format (either `hmac_jwkset` or `hmac_key`) at a time.
  :::


## JWT encoding and validation [jwt-validation]

JWTs can be parsed into three pieces:

Header
:   Provides information about how to validate the token.

Claims
:   Contains data about the calling user or application.

Signature
:   The data that’s used to validate the token.

```js
Header: {"typ":"JWT","alg":"HS256"}
Claims: {"aud":"aud8","sub":"security_test_user","iss":"iss8","exp":4070908800,"iat":946684800}
Signature: UnnFmsoFKfNmKMsVoDQmKI_3-j95PCaKdgqqau3jPMY
```

This example illustrates a partial decoding of a JWT. The validity period is from 2000 to 2099 (inclusive), as defined by the issue time (`iat`) and expiration time (`exp`). JWTs typically have a validity period shorter than 100 years, such as 1-2 hours or 1-7 days, not an entire human life.

The signature in this example is deterministic because the header, claims, and HMAC key are fixed. JWTs typically have a `nonce` claim to make the signature non-deterministic. The supported JWT encoding is JSON Web Signature (JWS), and the JWS `Header` and `Signature` are validated using OpenID Connect ID Token validation rules. Some validation is customizable through [JWT realm settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-jwt-settings).

### Header claims [jwt-validation-header]

The header claims indicate the token type and the algorithm used to sign the token.

`alg`
:   (Required, String) Indicates the algorithm that was used to sign the token, such as `HS256`. The algorithm must be in the realm’s allow list.

`typ`
:   (Optional, String) Indicates the token type, which must be `JWT`.


### Payload claims [jwt-validation-payload]

Tokens contain several claims, which provide information about the user who is issuing the token, and the token itself. Depending on the token type, these information can optionally be identified by different claims.

#### JWT payload claims [_jwt_payload_claims]

The following claims are validated by a subset of OIDC ID token rules.

{{es}} doesn’t validate `nonce` claims, but a custom JWT issuer can add a random `nonce` claim to introduce entropy into the signature.

::::{note}
You can relax validation of any of the time-based claims by setting `allowed_clock_skew`. This value sets the maximum allowed clock skew before validating JWTs with respect to their authentication time (`auth_time`), creation (`iat`), not before (`nbf`), and expiration times (`exp`).
::::


`iss`
:   (Required, String) Denotes the issuer that created the ID token. The value must be an exact, case-sensitive match to the value in the `allowed_issuer` setting.

`sub`
:   (Required*, String) Indicates the subject that the ID token is created for. If the JWT realm is of the `id_token` type, this claim is mandatory. A JWT realm of the `id_token` type by defaults accepts all subjects. A JWT realm of the access_token type must specify the `allowed_subjects` setting and the subject value must be an exact, case-sensitive match to any of the CSV values in the allowed_subjects setting. A JWT realm of the access_token type can specify a fallback claim that will be used in place where the `sub` claim does not exist.

`aud`
:   (Required*, String) Indicates the audiences that the ID token is for, expressed as a comma-separated value (CSV). One of the values must be an exact, case-sensitive match to any of the CSV values in the `allowed_audiences` setting. If the JWT realm is of the `id_token` type, this claim is mandatory. A JWT realm of the `access_token` type can specify a fallback claim that will be used in place where the `aud` claim does not exist.

`exp`
:   (Required, integer) Expiration time for the ID token, expressed in UTC seconds since epoch.

`iat`
:   (Required, integer) Time that the ID token was issued, expressed in UTC seconds since epoch.

`nbf`
:   (Optional, integer) Indicates the time before which the JWT must not be accepted, expressed as UTC seconds since epoch. This claim is optional. If it exists, a JWT realm of `id_token` type will verify it, while a JWT realm of `access_token` will just ignore it.

`auth_time`
:   (Optional, integer) Time when the user authenticated to the JWT issuer, expressed as UTC seconds since epoch. This claim is optional. If it exists, a JWT realm of `id_token` type will verify it, while a JWT realm of `access_token` will just ignore it.


#### {{es}} settings for consuming JWT claims [jwt-validation-payload-es]

{{es}} uses JWT claims for the following settings.

`principal`
:   (Required, String) Contains the user’s principal (username). The value is configurable using the realm setting `claims.principal`. You can configure an optional regular expression using the `claim_patterns.principal` to extract a substring.

`groups`
:   (Optional, JSON array) Contains the user’s group membership. The value is configurable using the realm setting `claims.groups`. You can configure an optional regular expression using the realm setting `claim_patterns.groups` to extract a substring value.

`name`
:   (Optional, String) Contains a human-readable identifier that identifies the subject of the token. The value is configurable using the realm setting `claims.name`. You can configure an optional regular expression using the realm setting `claim_patterns.name` to extract a substring value.

`mail`
:   (Optional, String) Contains the e-mail address to associate with the user. The value is configurable using the realm setting `claims.mail`. You can configure an optional regular expression using the realm setting `claim_patterns.mail` to extract a substring value.

`dn`
:   (Optional, String) Contains the user’s Distinguished Name (DN), which uniquely identifies a user or group. The value is configurable using the realm setting `claims.dn`. You can configure an optional regular expression using the realm setting `claim_patterns.dn` to extract a substring value.


## Role mapping [jwt-authorization]

You can map LDAP groups to roles in the following ways:

* Using the role mappings page in {{kib}}.
* Using the [role mapping API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-put-role-mapping).
* By delegating authorization [to another realm](#jwt-authorization-delegation).

For more information, see [Mapping users and groups to roles](/deploy-manage/users-roles/cluster-or-deployment-auth/mapping-users-groups-to-roles.md).

::::{important}
You can't map roles in the JWT realm using the `role_mapping.yml` file.
::::

### Authorizing with the role mapping API [jwt-authorization-role-mapping]

You can use the [create or update role mappings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-put-role-mapping) to define role mappings that determine which roles should be assigned to each user based on their username, groups, or other metadata.

```console
PUT /_security/role_mapping/jwt1_users?refresh=true <1>
{
  "roles" : [ "user" ], <2>
  "rules" : { "all" : [ <3>
      { "field": { "realm.name": "jwt1" } }, <4>
      { "field": { "username": "principalname1" } },
      { "field": { "dn": "CN=Principal Name 1,DC=example.com" } },
      { "field": { "groups": "group1" } },
      { "field": { "metadata.jwt_claim_other": "other1" } }
  ] },
  "enabled": true
}
```

1. The mapping name.
2. The {{stack}} role to map to.
3. A rule specifying the JWT role to map from.
4. `realm.name` can be any string containing only alphanumeric characters, underscores, and hyphens.

If you use this API in the JWT realm, the following claims are available for role mapping:

`principal`
:   (Required, String) Principal claim that is used as the {{es}} user’s username.

`dn`
:   (Optional, String) Distinguished Name (DN) that is used as the {{es}} user’s DN.

`groups`
:   (Optional, String) Comma-separated value (CSV) list that is used as the {{es}} user’s list of groups.

`metadata`
:   (Optional, object) Additional metadata about the user, such as strings, integers, boolean values, and collections that are used as the {{es}} user’s metadata. These values are key value pairs formatted as `metadata.jwt_claim_<key>` = `<value>`.


### Delegating JWT authorization to another realm [jwt-authorization-delegation]

If you [delegate authorization](../../../deploy-manage/users-roles/cluster-or-deployment-auth/realm-chains.md#authorization_realms) to other realms from the JWT realm, only the `principal` claim is available for role lookup. When delegating the assignment and lookup of roles to another realm from the JWT realm, claims for `dn`, `groups`, `mail`, `metadata`, and `name` are not used for the {{es}} user’s values. Only the JWT `principal` claim is passed to the delegated authorization realms. The realms that are delegated for authorization - not the JWT realm - become responsible for populating all of the {{es}} user’s values.

The following example shows how you define delegation authorization in the [`elasticsearch.yml`](/deploy-manage/stack-settings.md) file to multiple other realms from the JWT realm. A JWT realm named `jwt2` is delegating authorization to multiple realms:

```yaml
xpack.security.authc.realms.jwt.jwt2.authorization_realms: file1,native1,ldap1,ad1
```

You can then use the [create or update role mappings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-put-role-mapping) to map roles to the authorizing realm. The following example maps roles in the `native1` realm for the `principalname1` JWT principal.

```console
PUT /_security/role_mapping/native1_users?refresh=true
{
  "roles" : [ "user" ],
  "rules" : { "all" : [
      { "field": { "realm.name": "native1" } },
      { "field": { "username": "principalname1" } }
  ] },
  "enabled": true
}
```

If realm `jwt2` successfully authenticates a client with a JWT for principal `principalname1`, and delegates authorization to one of the listed realms (such as `native1`), then that realm can look up the {{es}} user’s values. With this defined role mapping, the realm can also look up this role mapping rule linked to realm `native1`.


## Applying the `run_as` privilege to JWT realm users [jwt-realm-runas]

{{es}} can retrieve roles for a JWT user through either role mapping or delegated authorization. Regardless of which option you choose, you can apply the [`run_as` privilege](../../../deploy-manage/users-roles/cluster-or-deployment-auth/submitting-requests-on-behalf-of-other-users.md#run-as-privilege-apply) to a role so that a user can submit authenticated requests to "run as" a different user. To submit requests as another user, include the `es-security-runas-user` header in your requests. Requests run as if they were issued from that user and {{es}} uses their roles.

For example, let’s assume that there’s a user with the username `user123_runas`. The following request creates a user role named `jwt_role1`, which specifies a `run_as` user with the `user123_runas` username. Any user with the `jwt_role1` role can issue requests as the specified `run_as` user.

```console
POST /_security/role/jwt_role1?refresh=true
{
  "cluster": ["manage"],
  "indices": [ { "names": [ "*" ], "privileges": ["read"] } ],
  "run_as": [ "user123_runas" ],
  "metadata" : { "version" : 1 }
}
```

You can then map that role to a user in a specific realm. The following request maps the `jwt_role1` role to a user with the username `user2` in the `jwt2` JWT realm. This means that {{es}} will use the `jwt2` realm to authenticate the user named `user2`. Because `user2` has a role (the `jwt_role1` role) that includes the `run_as` privilege, {{es}} retrieves the role mappings for the `user123_runas` user and uses the roles for that user to submit requests.

```console
POST /_security/role_mapping/jwt_user1?refresh=true
{
  "roles": [ "jwt_role1"],
  "rules" : { "all" : [
      { "field": { "realm.name": "jwt2" } },
      { "field": { "username": "user2" } }
  ] },
  "enabled": true,
  "metadata" : { "version" : 1 }
}
```

After mapping the roles, you can make an [authenticated call](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-authenticate) to {{es}} using a JWT and include the `ES-Client-Authentication` header:

$$$jwt-auth-shared-secret-scheme-example$$$

```sh
curl -s -X GET -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhdWQiOlsiZXMwMSIsImVzMDIiLCJlczAzIl0sInN1YiI6InVzZXIyIiwiaXNzIjoibXktaXNzdWVyIiwiZXhwIjo0MDcwOTA4ODAwLCJpYXQiOjk0NjY4NDgwMCwiZW1haWwiOiJ1c2VyMkBzb21ldGhpbmcuZXhhbXBsZS5jb20ifQ.UgO_9w--EoRyUKcWM5xh9SimTfMzl1aVu6ZBsRWhxQA" -H "ES-Client-Authentication: sharedsecret test-secret" https://localhost:9200/_security/_authenticate
```

The response includes the user who submitted the request (`user2`), including the `jwt_role1` role that you mapped to this user in the JWT realm:

```sh
{"username":"user2","roles":["jwt_role1"],"full_name":null,"email":"user2@something.example.com",
"metadata":{"jwt_claim_email":"user2@something.example.com","jwt_claim_aud":["es01","es02","es03"],
"jwt_claim_sub":"user2","jwt_claim_iss":"my-issuer"},"enabled":true,"authentication_realm":
{"name":"jwt2","type":"jwt"},"lookup_realm":{"name":"jwt2","type":"jwt"},"authentication_type":"realm"}
%
```

If you want to specify a request as the `run_as` user, include the `es-security-runas-user` header with the name of the user that you want to submit requests as. The following request uses the `user123_runas` user:

```sh
curl -s -X GET -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhdWQiOlsiZXMwMSIsImVzMDIiLCJlczAzIl0sInN1YiI6InVzZXIyIiwiaXNzIjoibXktaXNzdWVyIiwiZXhwIjo0MDcwOTA4ODAwLCJpYXQiOjk0NjY4NDgwMCwiZW1haWwiOiJ1c2VyMkBzb21ldGhpbmcuZXhhbXBsZS5jb20ifQ.UgO_9w--EoRyUKcWM5xh9SimTfMzl1aVu6ZBsRWhxQA" -H "ES-Client-Authentication: sharedsecret test-secret" -H "es-security-runas-user: user123_runas" https://localhost:9200/_security/_authenticate
```

In the response, you’ll see that the `user123_runas` user submitted the request, and {{es}} used the `jwt_role1` role:

```sh
{"username":"user123_runas","roles":["jwt_role1"],"full_name":null,"email":null,"metadata":{},
"enabled":true,"authentication_realm":{"name":"jwt2","type":"jwt"},"lookup_realm":{"name":"native",
"type":"native"},"authentication_type":"realm"}%
```


## PKC JWKS reloading [jwt-realm-jwkset-reloading]

JWT authentication supports signature verification using PKC (Public Key Cryptography) or HMAC algorithms.

PKC JSON Web Token Key Sets (JWKS) can contain public RSA and EC keys. HMAC JWKS or an HMAC UTF-8 JWK contain secret keys. JWT issuers typically rotate PKC JWKS more frequently (such as daily), because RSA and EC public keys are designed to be easier to distribute than secret keys like HMAC.

JWT realms load a PKC JWKS and an HMAC JWKS or HMAC UTF-8 JWK at startup. JWT realms can also reload PKC JWKS contents at runtime; a reload is triggered by signature validation failures.

::::{note}
HMAC JWKS or HMAC UTF-8 JWK reloading is not supported at this time.
::::


Load failures, parse errors, and configuration errors prevent a node from starting (and restarting). However, runtime PKC reload errors and recoveries are handled gracefully.

All other JWT realm validations are checked before a signature failure can trigger a PKC JWKS reload. If multiple JWT authentication signature failures occur simultaneously with a single {{es}} node, reloads are combined to reduce the reloads that are sent externally.

Separate reload requests cannot be combined if JWT signature failures trigger:

* PKC JWKS reloads in different {{es}} nodes
* PKC JWKS reloads in the same {{es}} node at different times

::::{important}
Enabling client authentication (`client_authentication.type`) is strongly recommended. Only trusted client applications and realm-specific JWT users can trigger PKC reload attempts. Additionally, configuring the following [JWT security settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-jwt-settings) is recommended:

* `allowed_audiences`
* `allowed_clock_skew`
* `allowed_issuer`
* `allowed_signature_algorithms`

::::




## Authorizing to the JWT realm with an HMAC UTF-8 key [hmac-oidc-example]

The following settings are for a JWT issuer, {{es}}, and a client of {{es}}. The example HMAC key is in an OIDC format that’s compatible with HMAC. The key bytes are the UTF-8 encoding of the UNICODE characters.

::::{important}
HMAC UTF-8 keys need to be longer than HMAC random byte keys to achieve the same key strength.
::::


### JWT issuer [hmac-oidc-example-jwt-issuer]

The following values are for the bespoke JWT issuer.

```js
Issuer:     iss8
Audiences:  aud8
Algorithms: HS256
HMAC UTF-8: hmac-oidc-key-string-for-hs256-algorithm
```


### JWT realm settings [hmac-oidc-example-jwt-realm]

To define a JWT realm, add the following realm settings to [`elasticsearch.yml`](/deploy-manage/stack-settings.md).

```yaml
xpack.security.authc.realms.jwt.jwt8.order: 8 <1>
xpack.security.authc.realms.jwt.jwt8.allowed_issuer: iss8
xpack.security.authc.realms.jwt.jwt8.allowed_audiences: [aud8]
xpack.security.authc.realms.jwt.jwt8.allowed_signature_algorithms: [HS256]
xpack.security.authc.realms.jwt.jwt8.claims.principal: sub
xpack.security.authc.realms.jwt.jwt8.client_authentication.type: shared_secret
```

1. In {{ecloud}}, the realm order starts at `2`. `0` and `1` are reserved in the realm chain on {{ecloud}}.



### JWT realm secure settings [_jwt_realm_secure_settings]

After defining the realm settings, use the [`elasticsearch-keystore`](elasticsearch://reference/elasticsearch/command-line-tools/elasticsearch-keystore.md) tool to add the following secure settings to the {{es}} keystore. In {{ecloud}}, you define settings for the {{es}} keystore under **Security** in your deployment.

```yaml
xpack.security.authc.realms.jwt.jwt8.hmac_key: hmac-oidc-key-string-for-hs256-algorithm
xpack.security.authc.realms.jwt.jwt8.client_authentication.shared_secret: client-shared-secret-string
```


### JWT realm role mapping rule [_jwt_realm_role_mapping_rule]

The following request creates role mappings for {{es}} in the `jwt8` realm for the user `principalname1`:

```console
PUT /_security/role_mapping/jwt8_users?refresh=true
{
  "roles" : [ "user" ],
  "rules" : { "all" : [
      { "field": { "realm.name": "jwt8" } },
      { "field": { "username": "principalname1" } }
  ] },
  "enabled": true
}
```


### Request headers [hmac-oidc-example-request-headers]

The following header settings are for an {{es}} client.

```js
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJpc3M4IiwiYXVkIjoiYXVkOCIsInN1YiI6InNlY3VyaXR5X3Rlc3RfdXNlciIsImV4cCI6NDA3MDkwODgwMCwiaWF0Ijo5NDY2ODQ4MDB9.UnnFmsoFKfNmKMsVoDQmKI_3-j95PCaKdgqqau3jPMY
ES-Client-Authentication: SharedSecret client-shared-secret-string
```

You can use this header in a `curl` request to make an authenticated call to {{es}}. Both the bearer token and the client authorization token must be specified as separate headers with the `-H` option:

```sh
curl -s -X GET -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJpc3M4IiwiYXVkIjoiYXVkOCIsInN1YiI6InNlY3VyaXR5X3Rlc3RfdXNlciIsImV4cCI6NDA3MDkwODgwMCwiaWF0Ijo5NDY2ODQ4MDB9.UnnFmsoFKfNmKMsVoDQmKI_3-j95PCaKdgqqau3jPMY" -H "ES-Client-Authentication: SharedSecret client-shared-secret-string" https://localhost:9200/_security/_authenticate
```

If you used role mapping in the JWT realm, the response includes the user’s `username`, their `roles`, metadata about the user, and the details about the JWT realm itself.

```sh
{"username":"user2","roles":["jwt_role1"],"full_name":null,"email":"user2@something.example.com",
"metadata":{"jwt_claim_email":"user2@something.example.com","jwt_claim_aud":["es01","es02","es03"],
"jwt_claim_sub":"user2","jwt_claim_iss":"my-issuer"},"enabled":true,"authentication_realm":
{"name":"jwt2","type":"jwt"},"lookup_realm":{"name":"jwt2","type":"jwt"},"authentication_type":"realm"}
```
