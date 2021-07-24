## Build base components

Clone and build individual components:

### Hydra

```sh
mkdir -p $GOPATH/src/github.com/ory/hydra
cd $GOPATH/src/github.com/ory/hydra
git clone https://github.com/ory/hydra.git .
git checkout v1.10.3
docker build -t ory-hydra:v1.10.3 -f .docker/Dockerfile-build .
```

### Keto

```sh
mkdir -p $GOPATH/src/github.com/ory/keto
cd $GOPATH/src/github.com/ory/keto
git clone https://github.com/ory/keto.git .
git checkout v0.6.0-alpha.3
docker build -t ory-keto:v0.6.0-alpha.3 -f .docker/Dockerfile-build .
```

### Kratos

```sh
mkdir -p $GOPATH/src/github.com/ory/kratos
cd $GOPATH/src/github.com/ory/kratos
git clone https://github.com/ory/kratos.git .
git checkout v0.7.1-alpha.1
docker build -t ory-kratos:v0.7.1-alpha.1 -f .docker/Dockerfile-build .
```

### Oathkeeper

Right now, Oathkeeper is the only component not providing Docker based build:

```sh
mkdir -p $GOPATH/src/github.com/ory/oathkeeper
cd $GOPATH/src/github.com/ory/oathkeeper
git clone https://github.com/ory/oathkeeper.git .
git checkout v0.38.14-beta.1
docker build -t ory-kratos:v0.38.14-beta.1 -f Dockerfile-dc .
```

## Build additional components required by the Compose setup

### Kratos self service UI

This is an example browser facing application implementing login, registration, verification and link recovery flows:

```sh
mkdir -p $GOPATH/src/github.com/ory/kratos-selfservice-ui-node
cd $GOPATH/src/github.com/ory/kratos-selfservice-ui-node
git clone https://github.com/ory/kratos-selfservice-ui-node.git .
git checkout v0.7.1-alpha.1
docker build -t ory-kratos-selfservice-ui-node:v0.7.1-alpha.1 .
```

### Mailslurper

Kratos always sends emails and mailslurper is a thin SMTP server used by the Compose:

```
mkdir -p $GOPATH/src/github.com/ory/mailslurper
cd $GOPATH/src/github.com/ory/mailslurper
git clone https://github.com/ory/mailslurper.git .
git checkout master
docker build -t ory-mailslurper:master -f Dockerfile-smtps .
```

## Run

```sh
cd compose/
docker run --rm -ti ory-oathkeeper:v0.38.14-beta.1 credentials generate --alg RS256 > configs/oathkeeper/jwks.json
docker-compose -f compose.yml up
```

## Test individual components

### Hydra

Create an OAuth 2.0 Client:

```sh
docker-compose -f compose.yml exec hydra \
    hydra clients create \
    --endpoint http://127.0.0.1:4445/ \
    --id my-client \
    --secret secret \
    -g client_credentials
```

List clients:

```sh
docker-compose -f compose.yml exec hydra \
    hydra clients list \
    --endpoint http://127.0.0.1:4445/
```

Perform credentials grant:

```sh
docker-compose -f compose.yml exec hydra \
    hydra token client \
    --endpoint http://127.0.0.1:4444/ \
    --client-id my-client \
    --client-secret secret \
    --scope openid,offline
```

Copy the output access token and introspect:

```sh
docker-compose -f compose.yml exec hydra \
    hydra token introspect \
    --endpoint http://127.0.0.1:4445/ ...access token goes here...
```

### Keto

The namespaces are defined in the `compose/configs/keto/keto.yml` file.

Create a relation tuple:

```sh
curl -XPUT --data '{
  "namespace": "default-namespace",
  "object": "blog_posts:my-first-blog-post",
  "relation": "delete",
  "subject": "alice"
}' http://localhost:4467/relation-tuples
```

Check if the tuple has been created:

```sh
curl http://localhost:4466/relation-tuples?namespace=default-namespace
```

Check if user Alice is allowed to delete the blog post:

```sh
curl -XPOST --data '{
  "namespace": "default-namespace",
  "object": "blog_posts:my-first-blog-post",
  "relation": "delete",
  "subject": "alice"
}' http://localhost:4466/check
```

### Kratos

Verify that Kratos is up:

```sh
curl --silent http://localhost:4433/health/alive | jq '.'
```

The result should be:

```json
{
  "status": "ok"
}
```

Now, open `http://127.0.0.1:4455/dashboard` in the browser. You should see the following self service app UI:

![Self service Kratos UI](docs/aaa-setup-kratos-self-service-ui.png)

Register an account and sign in.

![Self service Kratos UI signed in user](docs/aaa-setup-kratos-signed-in.png)

### Oathkeeper

The rules are defined in `compose/configs/oathkeeper/rules.json` file. The `allow-anonymous-with-header-mutator` rule allows an unauthenticated access to the `http://127.0.0.1:4455/anything/header` URL. Here we validate that:

```sh
curl -X GET http://127.0.0.1:4455/anything/header
```

Gives:

```json
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Accept-Encoding": "gzip",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.54.0",
    "X-Amzn-Trace-Id": "Root=1-5f91f003-273d759c25c50cdd24be33c6",
    "X-User": "guest"
  },
  "json": null,
  "method": "GET",
  "origin": "172.22.0.1, 92.209.32.233",
  "url": "https://httpbin.org/anything/header/anything/header"
}
```

The `deny-anonymous` rule disallows anonymous access to the `http://127.0.0.1:4455/anything/deny` URL.

```sh
curl --silent -H "Accept: application/json" -X GET http://127.0.0.1:4455/anything/deny | jq '.'
```

Returns:

```json
{
  "error": {
    "code": 403,
    "status": "Forbidden",
    "message": "Access credentials are not sufficient to access this resource"
  }
}
```

## Scenario

This scenario combines `hydra`, `keto` and `oathkeeper` together to build an end to end example where `oathkeeper` provides a rule protecting an endpoint using OAuth2 token and a `keto` permission. This is a first iteration and requires quite some switching between the terminal and a browser.

### Create an OpenID Client

```sh
docker-compose -f compose.yml exec hydra \
    hydra clients create \
    --endpoint http://127.0.0.1:4445 \
    --id scenario-client \
    --secret secret \
    --grant-types authorization_code,refresh_token \
    --response-types code,id_token \
    --scope openid,offline \
    --callbacks http://127.0.0.1:5555/callback
```

#### List Clients

```sh
docker-compose -f compose.yml exec hydra \
    hydra clients list \
    --endpoint http://127.0.0.1:4445
```

### Request a user token

```sh
docker-compose -f compose.yml exec hydra \
    hydra token user \
    --client-id scenario-client \
    --client-secret secret \
    --endpoint http://127.0.0.1:4444/ \
    --port 5555 \
    --scope openid,offline
```

This will require navigating to `http://127.0.0.1:5555` in the browser.

![Exemplary OAuth 2.0 Consumer](docs/001-exemplary-oauth-2-consumer.png)

Click `Authorize application` link. The result is an empty page with a URL containing a `login_challenge` parameter.

![Authorize application](docs/002-login-challenge.png)

Copy the `login_challenge` value from the URL and proceed:

```sh
export login_challenge=...
```

```sh
curl --silent -XPUT http://localhost:4445/oauth2/auth/requests/login/accept?login_challenge=${login_challenge} -d '{
  "acr": "lol",
  "remember": false,
  "remember_for": 0,
  "subject": "scenario"
}' | jq '.'
```

The result is a JSON value similar to:

```json
{
  "redirect_to": "http://127.0.0.1:4444/oauth2/auth?audience=&client_id=scenario-client&login_verifier=fce3f73081244383a41b1776f0e8b259&max_age=0&nonce=ukpogmszferrybsixvvnmdyk&prompt=&redirect_uri=http%3A%2F%2F127.0.0.1%3A5555%2Fcallback&response_type=code&scope=openid+offline&state=zxzwdotymvehgmolwfujojsd"
}
```

Copy the value of `redirect_to` and navigate to that URL in the browser. You will be redirected to the `consent_challenge`.

![Consent challenge](docs/003-consent-challenge.png)

Copy the value of the `consent_challenge` and proceed:

```sh
export consent_challenge=...
```

```sh
curl --silent -XPUT http://localhost:4445/oauth2/auth/requests/consent/accept?consent_challenge=${consent_challenge} -d '{
  "grant_access_token_audience": [
    "scenario-client"
  ],
  "grant_scope": [
    "openid", "offline"
  ],
  "handled_at": "2020-10-23T20:49:00Z",
  "remember": false,
  "remember_for": 0,
  "session": {
    "id_token": {
      "first_name": "Ha",
      "last_name": "Hahah"
    }
  }
}' | jq '.'
```

The `session.id_token` property is how the additional claims can be passed to the ID token via user info. This behaviour is documented here: https://www.ory.sh/hydra/docs/concepts/openid-connect-oidc#userinfo. This is the basic primitive for integrating Kratos and Hydra.

The result will be another JSON like this:

```json
{
  "redirect_to": "http://127.0.0.1:4444/oauth2/auth?audience=&client_id=scenario-client&consent_verifier=154fc48c8aec46c79c8d0a27c1b7aab4&max_age=0&nonce=ukpogmszferrybsixvvnmdyk&prompt=&redirect_uri=http%3A%2F%2F127.0.0.1%3A5555%2Fcallback&response_type=code&scope=openid+offline&state=zxzwdotymvehgmolwfujojsd"
}
```

Again, copy the URL and navigate to it in the browser. You will now see the page with access token, refresh token and ID token.

![Tokens](docs/004-tokens.png)

Copy the value of the access token and run the final steps.

### Create the keto policy

```sh
curl -XPUT --data '{
  "namespace": "default-namespace",
  "object": "token:and:keto",
  "relation": "get",
  "subject": "scenario"
}' http://localhost:4467/relation-tuples
```

And validate the token - make sure you put the token in the command instead of `...access_token...`:

```sh
curl -H "Authorization: Bearer ...access_token..." http://127.0.0.1:4455/anything/token-and-keto
```

The result is similar to:

```json
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Accept-Encoding": "gzip",
    "Authorization": "Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzoxZjViOWYwMS0xZTk4LTQ0M2QtOGVjYi01M2RlYjI2NWYxZWIiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOlsic2NlbmFyaW8tY2xpZW50Il0sImNsaWVudF9pZCI6InNjZW5hcmlvLWNsaWVudCIsImV4cCI6MTYwMzQ5NTgyNSwiZXh0Ijp7fSwiaWF0IjoxNjAzNDkyMjI0LCJpc3MiOiJodHRwOi8vMTI3LjAuMC4xOjQ0NDQvIiwianRpIjoiMWMyMjYxMWQtOWRhNS00NmIwLWJkN2MtYzM3ZDk5ODhjOWIxIiwibmJmIjoxNjAzNDkyMjI0LCJzY3AiOlsib3BlbmlkIiwib2ZmbGluZSJdLCJzdWIiOiJzY2VuYXJpbyJ9.aaadR9WL-kAT7TFM9ra-loA6MOYJSgb_RqwgxDjKLMmQ3Cj16_jC3Pi7zKEgfwCS6npb_6mACFRkwCE9Ih5cApuiKjbQXd2pZNrGacywqYmHFD1H_lDT6z2wxEnVJFYDtaCPcufn7YsN0HbRyCNGAXX_eAJoMbp0pa0J8q3penukAEviphRztIzAV6gMrN45G8YM9DWjLsxw44luW2rbRH6F78AdugZkL78JYqIeSwn5Tu93XJ34buPzsdjaMOiyEV9xABgtLTfV3joOneT-yJYTAPj8CCGpxB3LCMTSvBzjiIN7eqcPcw84Kg6zE-cqWfnc3xyQojiFvypOdB-sJKI79n5D3mUdfAmx1fW_-6BoBDw2_nVaFHMS_yLyr9q5vK2FrOj7Cw6VeeoLbZYVREPM6QAdR7Jrleo-mheDS4XixThmmmRqTcduatYRXedTADoh57ICzSVOEJOZ0HP5o4OjUeTrXrm9yn7voxNbR6y-wrLdZZJ7SsQ498NrE72qKzZp-O8UmPWuKOeXt4MBPIVlmYmrjjQ0WseS_4yg3u6rvk80mb5EVaS8cN8dhuzIkHtv0OyOqccqWNVW57VdMza7reN2b0pd-JOmJayZHuO6tcEYsscjtxCZfIteoDiPEXXvKdAGG_8vdo0f53B7wRcEZTDKqqYTitY16G9qh8A",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.54.0",
    "X-Amzn-Trace-Id": "Root=1-5f935a2e-044927e62c7ddc6b1038475f"
  },
  "json": null,
  "method": "GET",
  "origin": "172.22.0.1, 92.209.32.233",
  "url": "https://httpbin.org/anything/token-and-keto/anything/token-and-keto"
}
```