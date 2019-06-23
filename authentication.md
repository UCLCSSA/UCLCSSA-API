# UCLCSSA Authentication

The current backend service is designed to serve only the WeChat miniprogram frontend. This implies several design considerations:

- **Users** can either be distinguished through their WeChat `unionid`, or distinguished by our custom `id`, which is one-to-one associated to a `unionid`. Since we do not have any plans for standalone apps, we are going to use the `unionid` as unique identifier.

Users will have to go through a two-step grant process in order to allow us to provide the full functionality, for WeChat and for UCLAPI.

1. **WeChat grant**. We will ask users for their permission to allow us to obtain their WeChat `unionid`, as well as the client-side provided `code` which we will use to request WeChat service to grant us a `session_key` for that user.
2. **UCLAPI grant**. We will also ask users for their permission to allow us to use their UCL-specific personal information so we can query information such as timetable information on their behalf via a personal `token`.

## Authentication Flow

A *user* can unlock full functionality by transition from `unauthenticated` -> `wechat-authenticated` -> `uclapi-authenticated`, upon which `uclapi-authenticated` status indicates full functionality.

### Unauthenticated -> WeChat-Authenticated Status

Upon entering the WeChat app, new users (`unauthenticated`) shall be prompted by the WeChat client to request their permission for us to obtain their personal information. This step is crucial for basic community-related functionality, and so is required from new users.

The WeChat client shall call the `wx.login()` ([wx.login](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/login/wx.login.html)) API. 

Upon user agreeing, the API returns the `success` response object consisting of

```typescript
{ code: string; }
```

With the `code` being a login token valid for 5 minutes. 

The WeChat client shall now send to the backend service at `POST /register/wechat` a request consisting of:

```typescript
{
  appId: string;
  appSecret: string;
  code: string;
}
```

Where `appId` and `appSecret` are obtained via registering the miniprogram with WeChat, and are available from the miniprogram.

The backend shall use this `code`, `appId` and `appSecret`, and send a request to the `auth.code2Session` API with the obtained information:

```typescript
'GET https://api.weixin.qq.com/sns/jscode2session'
	+ `?appid=${appId}`
	+ `&secret=${appSecret}`
	+ `&js_code=${code}`
	+ `&grant_type=authorization_code`
```

Upon a successful request to WeChat's `auth.code2Session` API endpoint, the response is returned consisting of

```typescript
{
  openid: string;
  session_key: string;
  unionid: string;
  errcode: number;
  errmsg: string;
}
```

With `errocode == 0` indicating a successful request. The backend shall now store `unionid` and associate with it the user’s `session_key`. A new UCLCSSA session key shall now be generated for our purposes, serving as the Authorization Bearer Token for our services. This token will be named `uclcssaSessionKey` and is returned to the WeChat client, which shall store it and use it in the HTTP `Authorization` header for future requests for `wechat-authenticated` users.

`wechat-authenticated` users shall be able to “logout” via `POST /logout` with

```typescript
{
  uclcssaSessionKey: string;
}
```

Which shall invalidate any session keys (generated, WeChat specific) associated with the user (the user’s `union_id` will still be retained for persisting his/her information such as posts).

`uclapi-authenticated` users who call `POST /logout` will have their UCLAPI token invalidated and must perform the authentication flow again.

###  WeChat-Authenticated -> UCLAPI-Authenticated

Only `wechat-authenticated` users shall be able to request an upgrade to their status to `uclapi-authenticated` (that is, for users to allow us to access their personal information on their behalf).

`wechat-authenticated` users who wish to upgrade to `uclapi-authenticated` status shall direct the WeChat client to send a confirmation email. This request is delegated to the backend, through the client calling `POST /register/uclapi` endpoint with

```typescript
{
  uclcssaSessionKey: string;
  email: string;
}
```

The `uclcssaSessionKey` here is only valid if the user has already logged in through the previous WeChat grant process. The `email` does not have to be UCL, since the user will have to login with their UCL credentials anyway.

The backend shall send a email to the user, containing a link to the endpoint `GET /authorise/uclapi?unionId={UNION_ID}` where `UNION_ID` is the user’s WeChat `unionid`. This API endpoint shall redirect the user to `GET https://uclapi.com/oauth/authorize` with query parameters `client_id` obtained from the backend’s environment variables and `state` being `UNION_ID` (this allows us to track which user is authorized).

Should the user agree to authorize the backend to retreive their information on their behalf by logging in through UCL with their UCL account, UCLAPI will call the callback URL provided. In this case, UCLAPI will call `POST /authorize/uclapi/callback` with

```typescript
{
  client_id: string;
  state: string;
  result: string;
  code: string;
}
```

Upon UCLAPI activating the callback, the backend shall check the `state` for matching `union_id` and use the `code`, `client_id` and `client_secret` (configured at backend), and send them to `GET https://uclapi.com/oauth/token` with the body:

```typescript
{
  code: string;
  client_id: string;
  client_secret: string;
}
```

Upon successful request, the response is

```json
{
  "scope": "[]",
  "state": "1",
  "ok": true,
  "client_id": "123.456",
  "token": "uclapi-user-abc-def-ghi-jkl"
}
```

The backend shall now associate the user’s `union_id` from `state` with the `token` and invalidate the `code`.