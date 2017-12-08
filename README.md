# The OAuth Flow

We uses OAuth 2.0's authorization code grant flow to issue access tokens on behalf of users.

Your web or mobile app should redirect users to the following URL:

```markdown
https://onescape.auth.ap-northeast-2.amazoncognito.com
```

## Step 1 - Sending users to authorize and/or install

The /oauth2/authorize endpoint signs the user in.

```markdown
GET /oauth2/authorize
```

The /oauth2/authorize endpoint only supports HTTPS GET. The user pool client typically makes this request through the system browser, which would typically be Custom Chrome Tab in Android and Safari View Control in iOS.

The following values should be passed as GET parameters:

- response_type (Required): The response type. Must be code or token. Indicates whether the client wants an authorization code (authorization code grant flow) for the end user or directly issues tokens for end user (implicit flow).
- client_id (Required): The Client ID. Must be a pre-registered client in the user pool and must be enabled for federation.
- redirect_uri (Required): The URL to which the authentication server redirects the browser after authorization has been granted by the user. Must have been pre-registered with a client.
- state (Optional but strongly recommended): An opaque value the clients adds to the initial request. The authorization server includes this value when redirecting back to the client. This value must be used by the client to prevent CSRF attacks.

## Step 2 - Users are redirected to your server with a verification code

If the user authorizes your app, We will redirect back to your specified redirect_uri with a temporary code in a code GET parameter, as well as a state parameter if you provided one in the previous step. If the states don't match, the request may have been created by a third party and you should abort the process.

Authorization codes may only be exchanged once and expire 10 minutes after issuance.

### Examples of Positive Requests

Sample Request: Authorization code grant request

```markdown
GET https://onescape.auth.ap-northeast-2.amazoncognito.com/oauth2/authorize?
response_type=code&
client_id=CLIENT_ID&
redirect_uri=https://REDIRECT_URI&
state=STATE&
```

Sample Response

The authentication server redirects back to your app with the authorization code and state. The code and state must be returned in the query string parameters and not in the fragment. A query string is the part of a web request that appears after a '?' character; the string can contain one or more parameters separated by '&' characters. A fragment is the part of a web request that appears after a '#' character to specify a subsection of a document.

```markdown
HTTP/1.1 302 Found
Location: https://REDIRECT_URI?code=AUTHORIZATION_CODE&state=STATE
```

### Examples of Negative Requests

The following are examples of negative requests:

If client_id and redirect_uri are valid but there are other problems with the request parameters (for example, if response_type is not included; if code_challenge is supplied but code_challenge_method is not supplied; or if code_challenge_method is not 'S256'), the authentication server redirects the error to client's redirect_uri.

```markdown
HTTP 1.1 302 Found Location: https://REDIRECT_URI?error=invalid_request
```

If the client requests 'code' or 'token' in response_type but does not have permission for these requests, the Amazon Cognito authorization server should return unauthorized_client to client's redirect_uri, as follows:
```markdown
HTTP 1.1 302 Found Location: https://REDIRECT_URI?error=unauthorized_client
```

If the client requests invalid, unknown, malformed scope, the Amazon Cognito authorization server should return invalid_scope to the client's redirect_uri, as follows:
```markdown
HTTP 1.1 302 Found Location: https://REDIRECT_URI?error=invalid_scope
```

If there is any unexpected error in the server, the authentication server should return server_error to client's redirect_uri. It should not be the HTTP 500 error displayed to the end user in the browser, because this error doesn't get sent to the client. The following error should return:
```markdown
HTTP 1.1 302 Found Location: https://REDIRECT_URI?error=server_error
```

## Step 3 - Exchanging a verification code for an access token

The /oauth2/token endpoint gets the user's tokens.

```markdown
POST /oauth2/token
```

The /oauth2/token endpoint only supports HTTPS POST. The user pool client makes requests to this endpoint directly and not through the system browser.

Request Parameters

```markdown
- Authorization: If the client was issued a secret, the client must pass its client_id and client_secret in the authorization header through Basic HTTP authorization. The secret is Basic Base64Encode(client_id:client_secret).
- code: Required if grant_type is authorization_code.
- Content-Type: Must always be 'application/x-www-form-urlencoded'.
```

Request Parameters in Body

- grant_type (Required): Grant type. Must be authorization_code or refresh_token
- client_id (Required): Client ID. Must be a preregistered client in the user pool. The client must be enabled for Amazon Cognito federation. 
- redirect_uri (Required only if grant_type is authorization_code): Must be the same redirect_uri that was used to get authorization_code in /oauth2/authorize.
- refresh_token (Required if grant_type is refresh_token): The refresh token.

### Examples of Positive Requests

Sample Request: Exchanging authorization code for tokens

```markdown
POST https://onescape.auth.ap-northeast-2.amazoncognito.com/oauth2/token >
Content-Type='application/x-www-form-urlencoded'&
Authorization=Basic ENCODED_CLIENT_ID_AND_CLIENT_SECRET

grant_type=authorization_code&
client_id=CLIENT_ID&
code=AUTHORIZATION_CODE&
redirect_uri=https://REDIRECT_URI
```

Sample response

```markdown
HTTP/1.1 200 OK
Content-Type: application/json

{
    "access_token": "eyJraWQiOiI5ZmRld1lYQ1owOGNmQ21BNmNoeGRWUVNCaWxiYzY2dmxiZkpoMGR0OHprPSIsImFsZyI6IlJTMjU2In0.eyJzdWIiOiI4MjI0MTVkMy1mYzdmLTQ5MzQtOWU1NS05ZGVjNjVlNzIxNDgiLCJ0b2tlbl91c2UiOiJhY2Nlc3MiLCJzY29wZSI6ImFwaVwvcGV0cyIsImlzcyI6Imh0dHBzOlwvXC9jb2duaXRvLWlkcC5hcC1ub3J0aGVhc3QtMi5hbWF6b25hd3MuY29tXC9hcC1ub3J0aGVhc3QtMl9qdERPaWFKNzgiLCJleHAiOjE1MTI2MTgyMzksImlhdCI6MTUxMjYxNDYzOSwidmVyc2lvbiI6MiwianRpIjoiNTUxMjkyMWYtNDgxZC00MGYyLTlkODMtODMxMzIyYWY5NzYxIiwiY2xpZW50X2lkIjoiNXVyamNpc2JmbnU4MmpoZDhmOWppY2wyYmkiLCJ1c2VybmFtZSI6ImEifQ.PFufmI2PT_9ziHlLSiT_FaMcJY-mewRtQn4NADtI-UrIIQ386NDlL1GZqufjOyV9Ps3xWJzdzrOBW_FEglgy2IklNvZjHMovI0EAOigw6OPjFxhXFHwGEZMkHyRqpPfutYBFylEoKQkowWypZpjhA3s9mo8nYUbKkTVj5ZaCm4xQ2lzNxv4N2ZYpCcWTUS4nHdb1qXHKO2qtpf1WLPG0SddPuNhqFQ3B9QwRYlfblaPo1s0PfJzB37HKtOZQVg2Ctu73ZpYFrsU4nVGELuaGV-zRWyfVFYv5MAYF4h7aCz0sK2GAb1t214xgwvcjzLD3-CwpY9WxIHLysTMeXmYwcg",
    "refresh_token": "eyJjdHkiOiJKV1QiLCJlbmMiOiJBMjU2R0NNIiwiYWxnIjoiUlNBLU9BRVAifQ.QGu6GAO8xxN1EQKiXTKcQJ3mCGYaQ2g3LntBeogocBOEgOAUialAXdj4xpEuldM9Wz7nzQrlK8TbaucMZH-jJX0c5br7xM0AyWVOQ2PHskNKn7-_XSjR5V0NwaEI14ljqKnN0V389S2M9j5uOjk3_zSKtFepE01bKbadeGqUsS6BEgD9yP0lGL5zvPpULp73rnjQQXGRiFFfH4RlOk9A7_lThLYWT8dW9VSgeQ9vZjGfao7_v3zrsuZsJrv7j7Gz09tsCVf2KWkQurPcnL-L6QPHEOUnkKuiu8pMjJsbF7EV0KiidWMOThkCcpewExoboCwbBr0k8uWxu9lteg-cBg.p_4ZQs7ptyZY9a64.oduxHhZtdZnjfPs_XGLI0Gh04_b1UCGk3Z0SmfSg7QMVVXH4KcncRsszkS0sxd5tp90kvwR5ZZBb4-icLeUQwTTub7_i0FWe6YvnbyhFT79XOblPQLoqXAAFBm9PmpoTzZWz-8ap-wiuESWK1lO7tVtlak8U4ywi_gQU7-LCTYe7L0g0EwQr7YLMRGVwVajOLDx1OB2Y6H0DkhpwV3VRu-Iky0W8CaOliRrE6FTjFEj7TP8EZWCvlUHMoGbllx74lHXFi39nRd-ls2nl_5CKtI3qZTbj5vIXPTWMBAl68AU4R3YcUTmaQHYSbv2Gyk_NpLt_Gu-szoU8HUiTvfPc5TRsXABS76PhDXCWwg4VUA7iQkOn6CiA1LmTdZJOzbBWzDQB6dwohsXChACtgSShTp8qwIW8JCjpqVjAE4jW0D5B7xQ_uC9mQNXWXSxE2340IPuv16NpQmP-daNj0QcFaUbquVGYb6-m6cEVyzQh4DztPtXjResSToYT9_SW-6swccwSUI34ENABUZDAR4_o-Y26x67253GOW2y6LKmKhnAAqEuIljD-muAaa6WvPFX5-w-qlszRN28H2LyzA4KXEDHN7BobzfKjJtuAptKL19TSRtkVkPoRwa6nPJjEHLEem3usAmci-SLy9mg1xS25PdOwabV7LK8_x4TY74ylBvBFib006W8DI3V4gYxAdwKL6Unfe2AUyIXnopFzn5PBg99P2zcAwUqnsyOZ3n8-MAcmvWldWMH870GnW58iqsg69ZvrXXEcQMsbMOlUCsVL_ZZzlXi_GTjdtutF2REU8icNq3pkhcjTurDkizydDZG6ONnMwVm1E8z0gwJmZZE42QnEmyJHOhXrsuoLceJOwlADxFP8MOnS-51ffEpB2anPxlxITVdzOH6rBX1h3mvCY5t_o5HiOyCk6eeuMjXk_oasTrI5srh4GBtIul9-_355Abmo8ici1xSmXldm8fkXG4xBvKtdyOBZNbsIr-IC-egB29qE0pS3e8P7P0vg63xu891nLoTv5wm1sEnKLOEEMhPOkT5RZpxKI0T33v_7BhCsRZc5h05SA6rt8c0rksCznuratgRfHx8zwA03nQkTRlfmKbCfW8nn1hNXERgLP4iOuGNtu2aK-alcZswF1Jk.XbvRu6yw9oE2mjVF2OBmIw",
    "expires_in": 3600,
    "token_type": "Bearer"
}
```

Sample Request: Exchanging refresh token for tokens

```markdown
POST https://onescape.auth.ap-northeast-2.amazoncognito.com/oauth2/token >
Content-Type='application/x-www-form-urlencoded'
Authorization=Basic ENCODED_CLIENT

grant_type=refresh_token&
client_id=CLIENT_ID&
refresh_token=REFRESH_TOKEN
```

Sample Response

```markdown
HTTP/1.1 200 OK
Content-Type: application/json

{
    "access_token": "eyJraWQiOiI5ZmRld1lYQ1owOGNmQ21BNmNoeGRWUVNCaWxiYzY2dmxiZkpoMGR0OHprPSIsImFsZyI6IlJTMjU2In0.eyJzdWIiOiI4MjI0MTVkMy1mYzdmLTQ5MzQtOWU1NS05ZGVjNjVlNzIxNDgiLCJ0b2tlbl91c2UiOiJhY2Nlc3MiLCJzY29wZSI6ImFwaVwvcGV0cyIsImlzcyI6Imh0dHBzOlwvXC9jb2duaXRvLWlkcC5hcC1ub3J0aGVhc3QtMi5hbWF6b25hd3MuY29tXC9hcC1ub3J0aGVhc3QtMl9qdERPaWFKNzgiLCJleHAiOjE1MTI2Mzk2NjAsImlhdCI6MTUxMjYzNjA2MCwidmVyc2lvbiI6MiwianRpIjoiOWQzZTA5OGUtNTRjNy00YmRkLWJjMGUtOTgxYzIzMTFiYzc0IiwiY2xpZW50X2lkIjoiNXVyamNpc2JmbnU4MmpoZDhmOWppY2wyYmkiLCJ1c2VybmFtZSI6ImEifQ.cFg-J-zgIu9s2XIl5vWNcRonWLVP1j6xj9387YJBoCuCssUpwGDThjr6C7oSJ5xMqPQoxP-mzYUOuVqAMltahaTokC4V5eJnHrwjpnZ1sz7xfrsUrrDvY0JNRX1MUJ8rpPuLI7en8KN0O264CJHU_VZlQrckmNV4QnQ4lvYpJof5OfCHRrSnJz5kw01Lcj-zO4BZUo41B7FCZWpNqtJ9N0Gw6URyLc-ytIeMu7TxoFZTiNhOwsnt6QPPXv6xrJ39HrxVs_74Vmdv52V40clCLkX7T3q3nji0EMsK2jt-TdGeRVqAoS9X3ZS-iXW7ZQLgq4iSBhRUzkaBdmDTsxwing",
    "expires_in": 3600,
    "token_type": "Bearer"
}
```

### Examples of Negative Requests

Sample Error Response

```markdown
HTTP/1.1 400 Bad Request
Content-Type: application/json;charset=UTF-8

{
"error":"invalid_request|invalid_client|invalid_grant|unauthorized_client|unsupported_grant_type|"
}
```

- invalid_request: The request is missing a required parameter, includes an unsupported parameter value (other than unsupported_grant_type), or is otherwise malformed. For example, grant_type is refresh_token but refresh_token is not included.
- invalid_client: Client authentication failed. For example, when the client includes client_id and client_secret in the authorization header, but there's no such client with that client_id and client_secret.
- invalid_grant: Refresh token has been revoked. Authorization code has been consumed already or does not exist.
- unauthorized_client: Client is not allowed for code grant flow or for refreshing tokens.
- unsupported_grant_type: Returned if grant_type is anything other than authorization_code or refresh_token.

# Using access tokens

The tokens awarded to your app can be used in requests to the API.

```markdown
https://61z5hoawji.execute-api.ap-northeast-2.amazonaws.com/v1
```

The best way to communicate your access tokens, also known as bearer tokens, is by presenting them in a request's Authorization HTTP header:

```markdown
GET /RESOURCE_NAME
Authorization: Bearer ACCESS_TOKEN
```

This approach is required when using application/json with a write method.

![alt text](https://github.com/onescape/oauth/blob/master/swaggerhub.jpeg?raw=true)

![alt text](https://github.com/onescape/oauth/blob/master/postman.jpeg?raw=true)

## Solarzon API

The Solarzon API is an interface for querying information from and enacting change in a Solarzon device.

You can check a document to the following URL:

```markdown
https://swaggerhub.com/apis/onescape/solarzon
```
