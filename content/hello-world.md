# Hello, World!

## Pre-requisites: Create a token to access the Red Hat registry

You will need to create a token to be able to fetch APIcast from the Red Hat registry. Go to [access.redhat.com/terms-based-registry](https://access.redhat.com/terms-based-registry/), log in with your developer account (if you have not already done so), and click "New Service Account."

Give the token a name (for the rest of this article, we will use "3scale") and a meaningful description.

Click "Create" and the generated token is displayed. Save the username and the token in a safe place for future reference.

Click the "OpenShift Secret" tab and then "3scale-secret.yaml" to download your token in a format OpenShift will understand. Save it somewhere convenient for later use.

![Download OpenShift Secret](/hello-world/download-openshift-secret.png)

Click the "Docker Login" tab and copy the "docker login" command somewhere convenient for later use.

![Copy/paste the Docker login command](/hello-world/docker-login.png)

## Deploy APIcast on OpenShift

To install APIcast, you will need an OpenShift instance. If your company has one, use it. If not, we recommend using [Red Hat Container Development Kit (CDK)/minishift](https://developers.redhat.com/products/cdk/hello-world/). Minishift is an OpenShift installation targeted at developers that runs on your laptop. If you need to install CDK/minishift, see [these instructions](https://developers.redhat.com/products/cdk/hello-world/).

Spin up a minishift instance:

```raw
$ minishift start
```

Create a new project for your APIcast trial:

```raw
$ oc new-project 3scale
```

Inject the token you downloaded in the "Pre-requisites" section in your OpenShift project, as a secret:

```raw
$ oc create -f ~/Downloads/*_3scale-secret.yaml
```

Find the name of your secret:

```raw
$ oc get secret
NAME                          TYPE                                  DATA      AGE
10072637-3scale-pull-secret   kubernetes.io/dockerconfigjson        1         3m
```

If you named your token "3scale" as suggested above, your secret should end with "-3scale-pull-secret." In this example, my secret is named "10072637-3scale-pull-secret."

Link your token with the default service account so that any pod in this project can use it (do not forget to change "10072637-sso-pull-secret" to your token name):

```raw
$ oc secrets link default 10072637-sso-pull-secret --for=pull
```

Import the APIcast ImageStream:
```raw
$ oc create -f https://raw.githubusercontent.com/3scale/3scale-amp-openshift-templates/2.4.0.GA/3scale-image-streams.yml
```

Import the OpenShift template:

```raw
$ oc create -f https://raw.githubusercontent.com/3scale/3scale-amp-openshift-templates/2.4.0.GA/apicast-gateway/apicast.yml
```



### Deploy APIcast on Docker

TODO