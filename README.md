# Web Api Template

A REST web api server template in Haskell.

This serves as a base to create API projects that comes with the following features to avoid repeated bolierplate in each one.

- Light weight & super fast Warp http server
- Supports HTTP 1.1 & HTTP/2 TLS
- JSON as input/output serialization using Aeson library
- Structured logging (JSON)
- Define the REST interface using Servant library
- JWT (Bearer) token based authentication
- Application config via ENVIRONMENT variables (via dotenv & envy)
- Health & Info endpoints
- Prometheus metrics endpoint

## TODO

- [x] Integrate with RIO
- [x] Integrate with Servant
- [x] Integrate with FastLogger
- [x] Integrate with doenv & envy
- [x] Integrate with Prometheus
- [x] Integrate with wai-util
- [x] Setup JWT Authentication
- [x] JSON Error formatting
- [x] Setup HTTPS
- [ ] Setup PSQL/SqlLite pool with Presistent
- [ ] Setup Stack template

## Getting started

To create a bare minimum API service all you need is below:

```haskell
#!/usr/bin/env stack
{- stack --resolver lts-14.27 runghc
 --package chakra
-}
{-# LANGUAGE NoImplicitPrelude, OverloadedStrings, UnicodeSyntax, DataKinds, TypeOperators #-}
import Chakra

type HelloRoute = "hello" :> QueryParam "name" Text :> Get '[PlainText] Text
type API = HelloRoute :<|> EmptyAPI

hello :: Maybe Text -> BasicApp Text
hello name = do
  let name' = fromMaybe "Sensei!" name
  logInfo $ "Saying hello to " <> display name'
  return $ "Hello " <> name' <> "!"

main :: IO ()
main = do
  let infoDetail = InfoDetail "example" "dev" "0.1" "change me"
      appEnv = appEnvironment infoDetail
      appVer = appVersion infoDetail
      appAPI = Proxy :: Proxy API
      appServer = hello :<|> emptyServer
  logFunc <- buildLogger appEnv appVer
  middlewares <- chakraMiddlewares infoDetail
  runChakraAppWithMetrics
    middlewares
    EmptyContext
    (logFunc, infoDetail)
    appAPI
    appServer
```

## Build & Run

```bash
make build
PORT=3000 make run

open http://localhost:3000/health
```

_Endpoints_

Info: http://localhost:3000/info

Health: http://localhost:3000/health

Metrics: http://localhost:3000/metrics

_User REST Resource:_

GET http://localhost:3000/users        # list all users

GET http://localhost:3000/users/123    # fetch the given user by id

POST http://localhost:3000/users/-1    # create new user resource

DELETE http://localhost:3000/users/123 # Delete user by id

## Run tests

```bash
make test
```

## HTTPS Setup

Generate RootCA & localhost Public & Private key pair

Be sure to edit `certs/domains.ext` file if you need more DNS aliases before executing these commands.

```bash
openssl req -x509 -nodes -new -sha256 -days 1024 -newkey rsa:2048 -keyout "certs/RootCA.key" -out "certs/RootCA.pem" -subj "/C=US/CN=Localhost-Root-CA"
openssl x509 -outform pem -in "certs/RootCA.pem" -out "certs/RootCA.crt"

openssl req -new -nodes -newkey rsa:2048 -keyout "certs/localhost.key" -out "certs/localhost.csr" -subj "/C=US/ST=NoWhere/L=NoWhere/O=Localhost-Certificates/CN=localhost.local"
openssl x509 -req -sha256 -days 1024 -in "certs/localhost.csr" -CA "certs/RootCA.pem" -CAkey "certs/RootCA.key" -CAcreateserial -extfile "certs/domains.ext" -out "certs/localhost.crt"
```

Once you have these keys then to start the server in https run the below command:

```bash
make run-https

# or
/path/to/urapiexec --port 3443 --protocol http+tls --tlskey certs/localhost.key --tlscert certs/localhost.crt
```

open https://localhost:3443/health

## JWT Authentication

This template supports only authentication against standard JWT issued by Azure, Okta, Keybase & other OAuth JWT authentication providers.
However its easily customizable to authenticate against custom issued JWT token. This template does not issue a JWT token.

The validate any incoming JWT's we need the public keys (provider by JWT issuer) to verify the payload and also need to verify the audiences. 
These are configurable via ENV variables

```bash
JWK_PATH="secrets/jwk.sig"
JWK_AUDIENCES="api://some-resource-id"
```
