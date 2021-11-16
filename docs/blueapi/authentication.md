# Authentication
Blue uses API client credentials for authentication. You can generate your API credentials either from Ripple under "Tools > API Access Tokens", or Wave(Pro) under "Settings > API Access Tokens".

## Environment setup
Set the environment variables below if you are using [bluectl](https://github.com/alphauslabs/bluectl) or any of our supported client libraries.

=== "Ripple"
    ```sh
    ALPHAUS_CLIENT_ID={ripple-client-id}
    ALPHAUS_CLIENT_SECRET={ripple-client-secret}
    ```

=== "Wave(Pro)"
    ```sh
    ALPHAUS_CLIENT_ID={wave-client-id}
    ALPHAUS_CLIENT_SECRET={wave-client-secret}
    ALPHAUS_AUTH_URL=https://login.alphaus.cloud/access_token
    ```

You can use `bluectl` to validate. To install:
```sh
# Should work with Linux, MacOS, and Windows through WSL/2:
$ brew install alphauslabs/tap/bluectl
```

 Then run the following command:
```sh
$ bluectl whoami
```

If successful, it will output some information about the authenticated user.

At the moment, setting both Ripple and Wave(Pro) client credentials is not supported. If both are set, authentication will default to Ripple.

If you're using either `bluectl` or any of our supported client libraries, the authentication flow is as follows. First, it will look for the following environment variables:
```sh
ALPHAUS_CLIENT_ID
ALPHAUS_CLIENT_SECRET
```

The `ALPHAUS_AUTH_URL` environment variable is optional for Ripple. For Wave(Pro) users, this can be set to:
```sh
ALPHAUS_AUTH_URL=https://login.alphaus.cloud/access_token
```

In most cases, the environment variables above should be sufficient. If those are not set, it will then look for:
```sh
ALPHAUS_RIPPLE_CLIENT_ID
ALPHAUS_RIPPLE_CLIENT_SECRET
```

If those are not set, it will finally look for:
```sh
ALPHAUS_WAVE_CLIENT_ID
ALPHAUS_WAVE_CLIENT_SECRET
```

## Using bluectl
If you prefer to call the Blue API HTTP endpoints directly, you can use `bluectl` to generate the access token. Useful for scripting.
```sh
# Get access token for production:
$ bluectl access-token
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhd...

# You can use the command above to provide access tokens to your other commands.
# For example:
$ curl -H "Authorization: Bearer $(bluectl access-token)" \
  https://api.alphaus.cloud/m/blue/iam/v1/whoami | jq
{
  "id":"test",
  "parent":"MSP-xxxxxxx",
  "metadata":{}
}

# If you want to access our NEXT (BETA) environment, you can do:
$ curl -H "Authorization: Bearer $(bluectl access-token \
  --client-id $MY_CLIENT_ID_NEXT \
  --client-secret $MY_CLIENT_SECRET_NEXT --beta)" \
  https://apinext.alphaus.cloud/m/blue/iam/v1/whoami | jq
{
  "id":"test",
  "parent":"MSP-xxxxxxx",
  "metadata":{}
}
```

You can also use `bluectl` to provide access tokens to our current, non-Blue APIs [here](https://docs.mobingi.com/v/api-reference/). For example:
```sh
$ curl -H "Authorization: Bearer $(bluectl access-token)" \
  https://api.alphaus.cloud/m/ripple/user | jq
{
  ...
}
```

## Access token endpoints
The following are the endpoints used to acquire product-specific access tokens. You will then use these tokens in your calls to the API using the `Authorization: Bearer {token}` HTTP header. Access tokens are not compatible between products. Ripple access tokens can only be used for Ripple endpoints; Wave(Pro) access tokens are only valid on Wave(Pro) endpoints.

=== "Ripple"
    ```
    https://login.alphaus.cloud/ripple/access_token
    ```

=== "Wave(Pro)"
    ```
    https://login.alphaus.cloud/access_token
    ```

To obtain an access token, send a POST message to the access token endpoint using the format described below.

**Request**

```
POST {access-token-url} HTTP1.1
Content-Type: multipart/form-data

{body formdata}
```

| **Name** | **Value** |
|---|---|
| `grant_type` | Valid value(s): `password`, `client_credentials` |
| `client_id` | The client id you received from Alphaus or from API. |
| `client_secret` | The client secret you received from Alphaus or from API. |
| `username` | You account username. Required if `grant_type` is set to `password`. |
| `password` | You account password. Required if `grant_type` is set to `password`. |
| `scope` | Valid value(s): `openid` |

**Response**

```json
{
  "id_token": "eyJ0eXAiOiJKV1Q...",
  "token_type": "Bearer",
  "expires_in": 86400,
  "access_token": "eyJ0eXAiOiJKV1Q...",
  "refresh_token": "def50200..."
}
```

**Example**

```sh
$ curl -X POST \
  -F client_id={client-id} \
  -F client_secret={client-secret} \
  -F grant_type=client_credentials \
  -F scope=openid \
  https://login.alphaus.cloud/ripple/access_token
```
