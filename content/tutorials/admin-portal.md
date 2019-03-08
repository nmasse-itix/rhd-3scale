# Use the 3scale Admin Portal to configure and manage APIcast

In this tutorial, you will connect your APIcast instance to your 3scale Admin
Portal and expose your first API.

As a pre-requisite, you need to [request a trial account on 3scale.net (it's free)](https://www.3scale.net/signup)!

## 1) Generate an Access Token for APIcast

Connect to the 3scale Admin Portal for which you signed up earlier. You can find your Admin Portal URL in the confirmation mail sent during signup. It looks like `https://TENANT-admin.3scale.net` where `TENANT` is the name you chose during signup.

- Click on the gear in the top right corner, go to **Personal** > **Tokens** and click **Add Access Token**.

*TODO Screenshot*

- Fill-in the name with `APIcast`
- Check the **Account Management API**
- Leave the default permission as **Read Only**
- Click **Create Access Token**

*TODO Screenshot*

- Copy the generated Access Token and store it a safe place! You will need it in the next part.
- Click **I have copied the token**

*TODO Screenshot*

## 2) Connect APIcast to the 3scale Admin Portal

Start APIcast in verbose mode to check if the connection between APIcast and the 3scale Admin Portal is established:

```sh
docker run -it --rm --name apicast -p 8080:8080 -e APICAST_CONFIGURATION_CACHE=300 \
           -e APICAST_CONFIGURATION_LOADER=boot -e THREESCALE_DEPLOYMENT_ENV=staging \
           -e THREESCALE_PORTAL_ENDPOINT=https://ACCESS_TOKEN@TENANT-admin.3scale.net \
           -e APICAST_LOG_LEVEL=info -e APICAST_RESPONSE_CODES=true \
           registry.redhat.io/3scale-amp24/apicast-gateway
```

You will need to replace `ACCESS_TOKEN` with the Access Token you generated
in the previous exercise and `TENANT` with the name of your tenant so that it
matches your 3scale Admin Portal URL.

In the last lines of the output, you should have something similar to:

```raw
2019/03/07 14:27:38 [info] 36#36: *26 [lua] configuration_store.lua:124: store(): added service 123456 configuration with hosts: api-789.production.gw.apicast.io, api-789.staging.gw.apicast.io ttl: 300, context: ngx.timer
```

If instead, you have such error message, double check the Access Token and Tenant are set correctly:

```raw
2019/03/07 14:21:00 [warn] 31#31: *2 [lua] remote_v2.lua:170: call(): failed to get list of services: invalid status: 403 (Forbidden) url: https://TENANT-admin.3scale.net/admin/api/services.json, context: ngx.timer
```

Hit `Ctrl-C` to stop APIcast.

You can now deploy the set of two APIcast instances that is required to use
3scale:

- one staging APIcast instance
- one production APIcast instance

Deploy a staging APIcast instance on port 8081:

```sh
docker run --rm -d --name apicast-staging -p 8081:8080 -e APICAST_CONFIGURATION_CACHE=0 \
           -e APICAST_CONFIGURATION_LOADER=lazy -e THREESCALE_DEPLOYMENT_ENV=staging \
           -e THREESCALE_PORTAL_ENDPOINT=https://ACCESS_TOKEN@TENANT-admin.3scale.net \
           -e APICAST_LOG_LEVEL=info -e APICAST_RESPONSE_CODES=true \
           registry.redhat.io/3scale-amp24/apicast-gateway
```

Deploy a production APIcast instance on port 8082:

```sh
docker run --rm -d --name apicast-production -p 8082:8080 -e APICAST_CONFIGURATION_CACHE=60 \
           -e APICAST_CONFIGURATION_LOADER=boot -e THREESCALE_DEPLOYMENT_ENV=production \
           -e THREESCALE_PORTAL_ENDPOINT=https://ACCESS_TOKEN@TENANT-admin.3scale.net \
           -e APICAST_LOG_LEVEL=warn -e APICAST_RESPONSE_CODES=true \
           registry.redhat.io/3scale-amp24/apicast-gateway
```

## 3) Deploy your first API

Connect to the 3scale Admin Portal for which you signed up earlier. You can find your Admin Portal URL in the confirmation mail sent during signup. It looks like `https://TENANT-admin.3scale.net` where `TENANT` is the name you chose during signup.

- In the dropdown list on the top side, select **Echo API**
- Go to **Integration** > **Configuration**
- Click **edit integration settings**

*TODO Screenshot*

- Select **APIcast self-managed**
- Scroll to the bottom and click **Update service**

*TODO Screenshot*

- Click **edit APIcast configuration**

*TODO Screenshot*

- Leave the Private Base URL to `http://echo-api.3scale.net:80`
- In the **Staging Public Base URL** field, type `http://localhost:8081`
- In the **Production Public Base URL** field, type `http://localhost:8082`

*TODO Screenshot*

- Scroll down and click **Update the Staging Environment**

*TODO Screenshot*

- Copy the `curl` command and paste it in a terminal (your `user_key` will be different from mine, this is normal):

```raw
$ curl "http://localhost:8081/echo?user_key=987654321"
{
  "method": "GET",
  "path": "/echo",
  "args": "user_key=987654321",
  "body": "",
  "headers": {
    "HTTP_VERSION": "HTTP/1.1",
    "HTTP_HOST": "echo-api.3scale.net",
    "HTTP_ACCEPT": "*/*",
    "HTTP_USER_AGENT": "curl/7.54.0",
    "HTTP_X_3SCALE_PROXY_SECRET_TOKEN": "Shared_secret_sent_from_proxy_to_API_backend_123456",
    "HTTP_X_REAL_IP": "172.17.0.1",
    "HTTP_X_FORWARDED_FOR": "10.0.103.54",
    "HTTP_X_FORWARDED_HOST": "echo-api.3scale.net",
    "HTTP_X_FORWARDED_PORT": "80",
    "HTTP_X_FORWARDED_PROTO": "http",
    "HTTP_FORWARDED": "for=10.0.103.54;host=echo-api.3scale.net;proto=http"
  },
  "uuid": "04b826af-4f69-4140-94ae-42c7181853be"
}
```

- Go back to **Integration** > **Configuration**
- Click on **Promote v.X to Production**

Wait one minute for the production APIcast to pickup changes in its
configuration and run again your `curl` command on port 8082 this time.
Your `user_key` will be different from mine, this is normal.

```raw
$ curl "http://localhost:8082/echo?user_key=987654321"
{
  "method": "GET",
  "path": "/echo",
  "args": "user_key=987654321",
  "body": "",
  "headers": {
    "HTTP_VERSION": "HTTP/1.1",
    "HTTP_HOST": "echo-api.3scale.net",
    "HTTP_ACCEPT": "*/*",
    "HTTP_USER_AGENT": "curl/7.54.0",
    "HTTP_X_3SCALE_PROXY_SECRET_TOKEN": "Shared_secret_sent_from_proxy_to_API_backend_123456",
    "HTTP_X_REAL_IP": "172.17.0.1",
    "HTTP_X_FORWARDED_FOR": "10.0.103.54",
    "HTTP_X_FORWARDED_HOST": "echo-api.3scale.net",
    "HTTP_X_FORWARDED_PORT": "80",
    "HTTP_X_FORWARDED_PROTO": "http",
    "HTTP_FORWARDED": "for=10.0.103.54;host=echo-api.3scale.net;proto=http"
  },
  "uuid": "04b826af-4f69-4140-94ae-42c7181853be"
}
```

**Congratulation, you just secured your first API with 3scale!**
