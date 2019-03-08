# Deploy APIcast

## 1) Get a token to access the Red Hat Registry

You will need to create a token to be able to fetch APIcast from the Red Hat registry. Go to [access.redhat.com/terms-based-registry](https://access.redhat.com/terms-based-registry/), log in with your developer account (if you have not already done so), and click "New Service Account."

Give the token a name (for the rest of this article, we will use "3scale") and a meaningful description.

Click "Create" and the generated token is displayed. Save the username and the token in a safe place for future reference.

Click the "Docker Login" tab and copy the "docker login" command somewhere convenient for later use.

![Copy/paste the Docker login command](/hello-world/docker-login.png)

Paste it in a terminal. This will log you in so that you can docker can pull the APIcast image.

If everything went fine, you should see something like this:

```raw
$ docker login -u='123456|3scale' -p=[REDACTED] registry.redhat.io
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Login Succeeded
```

## 2) Deploy APIcast as a standalone API Gateway

Create a configuration file for APIcast:

```json
cat > config.json <<EOF
{
  "services": [
    {
      "id": 1234,
      "backend_version": 1,
      "proxy": {
        "api_backend": "http://127.0.0.1:8081",
        "hostname_rewrite": "echo",
        "hosts": [ "localhost", "127.0.0.1" ],
        "credentials_location": "headers",
        "auth_user_key": "api-key",
        "policy_chain": [
          { "name": "apicast.policy.apicast" }
        ],
        "proxy_rules": [
          { "http_method": "GET", "pattern": "/", "metric_system_name": "hits", "delta": 1 }
        ]
      }
    }
  ]
}
EOF
```

Run APIcast in standalone mode:

```sh
docker run -it --rm --name apicast -p 8080:8080 -e APICAST_CONFIGURATION_CACHE=0 \
           -e APICAST_CONFIGURATION_LOADER=lazy -e APICAST_LOG_LEVEL=info \
           -v $PWD/config.json:/etc/apicast-config.json -e THREESCALE_CONFIG_FILE=/etc/apicast-config.json \
           registry.redhat.io/3scale-amp24/apicast-gateway
```

Confirm APIcast is working with:

```raw
$ curl -H "api-key: 123456" http://localhost:8080/echo
GET /test HTTP/1.1
X-Real-IP: 172.17.0.1
Host: echo
User-Agent: curl/7.54.0
Accept: */*
api-key: 123456
```

## 3) Protect your APIs with APIcast

Update the configuration file for APIcast:

```json
cat > config.json <<EOF
{
  "services": [
    {
      "id": 1234,
      "backend_version": 1,
      "proxy": {
        "api_backend": "http://echo-api.3scale.net",
        "hosts": [ "localhost", "127.0.0.1" ],
        "credentials_location": "headers",
        "auth_user_key": "api-key",
        "policy_chain": [
          { "name": "apicast.policy.apicast" }
        ],
        "proxy_rules": [
          { "http_method": "GET", "pattern": "/", "metric_system_name": "hits", "delta": 1 }
        ]
      }
    }
  ]
}
EOF
```

Confirm it works:

```raw
$ curl http://localhost:8080/test -H "api-key: 123456"
{
  "method": "GET",
  "path": "/test",
  "args": "",
  "body": "",
  "headers": {
    "HTTP_VERSION": "HTTP/1.1",
    "HTTP_HOST": "echo-api.3scale.net",
    "HTTP_ACCEPT": "*/*",
    "HTTP_API_KEY": "123456",
    "HTTP_USER_AGENT": "curl/7.54.0",
    "HTTP_X_REAL_IP": "172.17.0.1",
    "HTTP_X_FORWARDED_FOR": "90.79.1.247, 10.0.101.26",
    "HTTP_X_FORWARDED_HOST": "echo-api.3scale.net",
    "HTTP_X_FORWARDED_PORT": "80",
    "HTTP_X_FORWARDED_PROTO": "http",
    "HTTP_FORWARDED": "for=10.0.101.26;host=echo-api.3scale.net;proto=http"
  },
  "uuid": "d94aacc8-6a92-4b44-a5a3-94b05fa7e95b"
}
```
