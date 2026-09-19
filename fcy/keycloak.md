# Keycloak Configuration

OpenCloud uses the existing FCY Keycloak realm for authentication and its own
private LDAP service for provisioned directory records. A user is added to the
OpenCloud directory on first successful login.

## Realm

| Setting | Value |
| --- | --- |
| Keycloak host | `dev-id.fullcourseyellow.com` |
| Realm | `fullcourseyellow` |
| Issuer | `https://dev-id.fullcourseyellow.com/realms/fullcourseyellow` |
| Account page | `https://dev-id.fullcourseyellow.com/realms/fullcourseyellow/account` |

## Realm Roles

Create these realm roles:

| Keycloak role | OpenCloud role |
| --- | --- |
| `opencloudAdmin` | Full instance administration |
| `opencloudSpaceAdmin` | Space administration |
| `opencloudUser` | Standard user |
| `opencloudGuest` | Restricted user |

Assign exactly one OpenCloud role to every user who should access OpenCloud.
Users without a mapped role cannot log in. If multiple mapped roles are
assigned, OpenCloud selects the first matching role from its configured role
mapping, which can produce surprising privileges.

## Role Claim

Do not alter the realm's built-in `roles` client scope because other FCY clients
may depend on its current token structure. Create a dedicated client scope:

| Setting | Value |
| --- | --- |
| Name | `opencloud-roles` |
| Type | Default |
| Protocol | OpenID Connect |
| Include in token scope | Off |

Add a mapper to `opencloud-roles`:

| Mapper setting | Value |
| --- | --- |
| Name | `OpenCloud realm roles` |
| Mapper type | User Realm Role |
| Token claim name | `roles` |
| Claim JSON type | String |
| Multivalued | On |
| Add to ID token | On |
| Add to access token | On |
| Add to UserInfo | On |

On the client scope's **Scope** tab, disable **Full Scope Allowed** and add only
the four `opencloud*` realm roles. Attach `opencloud-roles` as a default client
scope to each OpenCloud client below. This produces the top-level multivalued
`roles` claim expected by `PROXY_ROLE_ASSIGNMENT_OIDC_CLAIM=roles` without
exposing unrelated realm roles.

## Common Client Settings

Create four separate OpenID Connect clients. Apply these settings to all four:

| Setting | Value |
| --- | --- |
| Client authentication | Off (public client) |
| Authorization | Off |
| Standard flow | On |
| Direct access grants | Off |
| Implicit flow | Off |
| PKCE method | `S256` |
| Default client scopes | `profile`, `email`, `opencloud-roles` |

No OpenCloud client secret is used. Desktop and mobile applications are public
clients and discover their client IDs and scopes through OpenCloud WebFinger.

## Web Client

| Setting | Value |
| --- | --- |
| Client ID | `web` |
| Root URL | `https://dev-cloud.fullcourseyellow.com` |
| Home URL | `https://dev-cloud.fullcourseyellow.com` |
| Valid redirect URI | `https://dev-cloud.fullcourseyellow.com/` |
| Valid redirect URI | `https://dev-cloud.fullcourseyellow.com/oidc-callback.html` |
| Valid redirect URI | `https://dev-cloud.fullcourseyellow.com/oidc-silent-redirect.html` |
| Valid post-logout redirect URI | `https://dev-cloud.fullcourseyellow.com/*` |
| Web origin | `https://dev-cloud.fullcourseyellow.com` |
| Backchannel logout URL | `https://dev-cloud.fullcourseyellow.com/backchannel_logout` |
| Backchannel logout session required | On |

## Desktop Client

| Setting | Value |
| --- | --- |
| Client ID | `OpenCloudDesktop` |
| Valid redirect URI | `http://127.0.0.1` |
| Valid redirect URI | `http://localhost` |
| Optional client scope | `offline_access` |

The loopback redirects are intentional. The desktop client opens a temporary
local callback listener after launching the browser-based Keycloak login.

## Android Client

| Setting | Value |
| --- | --- |
| Client ID | `OpenCloudAndroid` |
| Valid redirect URI | `oc://android.opencloud.eu` |
| Valid post-logout redirect URI | `oc://android.opencloud.eu` |
| Optional client scope | `offline_access` |

## iOS Client

| Setting | Value |
| --- | --- |
| Client ID | `OpenCloudIOS` |
| Valid redirect URI | `oc://ios.opencloud.eu` |
| Valid post-logout redirect URI | `oc://ios.opencloud.eu` |
| Optional client scope | `offline_access` |

## Optional Group Claim

To synchronize Keycloak group memberships, create another dedicated default
client scope with a **Group Membership** mapper. Emit a multivalued top-level
`groups` claim, disable the full group path unless it is needed, and attach the
scope to all four clients. Group claims are not required for initial login or
role assignment.

## Validation

After deployment, verify the issuer metadata:

```bash
curl --fail --silent --show-error \
  https://dev-id.fullcourseyellow.com/realms/fullcourseyellow/.well-known/openid-configuration
```

Verify desktop discovery through OpenCloud:

```bash
curl --fail --silent --show-error \
  'https://dev-cloud.fullcourseyellow.com/.well-known/webfinger?resource=https://dev-cloud.fullcourseyellow.com&rel=http://openid.net/specs/connect/1.0/issuer&platform=desktop'
```

The response must advertise issuer
`https://dev-id.fullcourseyellow.com/realms/fullcourseyellow`, client ID
`OpenCloudDesktop`, and the `offline_access` scope.

Finally, sign in with one user for each assigned role and inspect the OpenCloud
logs for provisioning or role-mapping errors:

```bash
docker compose logs opencloud
```
