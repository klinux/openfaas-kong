# Kong + OpenFaas
This repository will guide you to get Kong in front of OpenFaas.

## Install OpenFaas
Please, go to this repository and install OpenFaas, this repository use a PersistenVolumeClaim, ajust with your kubernetes configuration.

[faas-netes](https://github.com/klinux/faas-netes.git)

## Install Kong
After install OpenFaas, you can install Kong, this repository use the same namespace of OpenFaas.

Enter in yaml folder, inside this folter you will find some yaml files, install in this order:

```sh
kubectl create -f kong-db-dep.yml
```

```sh
kubectl create -f kong-migrations.yml
```

```sh
kubectl create -f kong-dep.yml
```

After Kong installation you can install the UI interface, [KONGA](https://github.com/pantsel/konga)

```sh
kubectl create -f konga-dep.yml
```

> Note: Ajust the Ingress sections on files, accordly your configurations.

## Configuring OpenFaas Endpoints inside Kong
You can do this task with two ways, command line or through KONGA web interface.

Here you can get with command line to configure endpoits.

### Configure "/function" services and routes

```sh
curl -k -X PUT \
    https://${INGRESS_ADMIN_URL}/services/function \
    -d 'url=http://gateway:8080/function'

curl -k -X POST \
    https://${INGRESS_ADMIN_URL}/services/function/routes \
    -d 'name=function' \
    -d 'protocols=http' \
    --data-urlencode 'paths[]=/function'
```

### Configure "/async-function" services and routes

```sh
curl -k -X PUT \
    https://${INGRESS_ADMIN_URL}/services/async-function \
    -d 'url=http://gateway:8080/async-function'

curl -k -X POST \
    https://${INGRESS_ADMIN_URL}/services/async-function/routes \
    -d 'name=async-function' \
    -d 'protocols=http' \
    --data-urlencode 'paths[]=/async-function'
```

> The -k options in curl is necessary if you are using self-signed certificates.

> Kong Admin API only supports authentication in enterprise edition, so, as best practice, don't publish kong adm Ingress, use KONGA to manage kong adm.

### Enable basic authentication
Kong support a lot of authentication methods, like basic, api key, oauth2, jwt etc.  
To simplify this deploy, we will use basic authentication.

### Enable basic plugin

```sh
curl -k -X POST https://${INGRESS_ADMIN_URL}/plugins \
    -d 'name=basic-auth' \
    -d 'config.hide_credentials=true'
```

### Create a cunsumer

```sh
curl -k https://${INGRESS_ADMIN_URL}/consumers/ -d 'username=Aladdin'
```

### Create a credential

```sh
curl -k -X POST https://${INGRESS_ADMIN_URL}/consumers/aladdin/basic-auth \
    -d 'username=Aladdin' \
    -d 'password=OpenSesame'
```

### Validate configuration

```sh
$ curl -k https://${INGRESS_PROXY_URL}/function/func_echoit -d 'hello world'
{"message": "Unauthorized"}

$ curl -k https://${INGRESS_PROXY_URL}/function/func_echoit -d 'hello world' \
-H 'Authorization: Basic xxxxxx'
{"message": "Invalid authentication credentials"}

$ echo -n aladdin:OpenSesame | base64
YWxhZGRpbjpPcGVuU2VzYW1l

$ curl -k https://${INGRESS_PROXY_URL}/function/func_echoit -d 'hello world' \
    -H 'Authorization: Basic YWxhZGRpbjpPcGVuU2VzYW1l'
hello world
```

### Configure the "/ui" endpoint, to access the UI of OpenFaas

```sh
curl -k -X PUT \
    https://${INGRESS_ADMIN_URL}/services/ui \
    -d 'url=http://gateway:8080/ui'

curl -k -X POST \
    https://${INGRESS_ADMIN_URL}/services/ui/routes \
    -d 'name=ui' \
    -d 'protocols=http' \
    --data-urlencode 'paths[]=/ui'
```

```sh

curl -k -X PUT \
    https://${INGRESS_ADMIN_URL}/services/system-functions \
    -d 'url=http://gateway:8080/system/functions'

curl -k -X POST \
    https://${INGRESS_ADMIN_URL}/services/system-functions/routes \
    -d 'name=system-functions' \
    -d 'protocols=http' \
    --data-urlencode 'paths[]=/system/functions'
```
### Test Configuration
Now visit https://${INGRESS_PROXY_URL}/ui/ in your browser where you will be asked for credentials.

## Add SSL
> This part, is the same OpenFaas doc [Original](https://github.com/openfaas/faas/blob/master/guide/kong_integration.md) 

Basic authentication does not protect from man in the middle attacks, so lets add SSL to encrypt the communication.

Create a cert. Here in the demo, we are creating selfsigned certs, but in production you should skip this step and use your existing certificates (or get some from Lets Encrypt).

```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/selfsigned.key -out /tmp/selfsigned.pem \
  -subj "/C=US/ST=CA/L=L/O=OrgName/OU=IT Department/CN=example.com"
```

### Add cert to Kong

```
kong_admin_curl -X POST http://localhost:8001/certificates \
    -F "cert=$(cat /tmp/selfsigned.pem)" \
    -F "key=$(cat /tmp/selfsigned.key)" \
    -F "snis=example.com"
HTTP/1.1 201 Created
...
```

### Put the cert in front OpenFaaS

```sh
kong_admin_curl -i -X POST http://localhost:8001/apis \
    -d "name=ssl-api" \
    -d "upstream_url=http://gateway:8080" \
    -d "hosts=example.com"
HTTP/1.1 201 Created
...
```

Verify that the cert is now in use. Note the '-k' parameter is just here to work around the fact that we are using self signed certs.

```sh
curl -k https://localhost:8443/function/func_echoit \
  -d 'hello world' -H 'Host: example.com '\
  -H 'Authorization: Basic YWxhZGRpbjpPcGVuU2VzYW1l'
hello world
```

## Configure your firewall

Between OpenFaaS and Kong a lot of ports are exposed on your host machine. Most importantly you should hide port 8080 since that is where OpenFaaS's functions live which you were trying to secure in the first place. In the end it is best to only expose either 8000 or 8443 out of your network depending if you added SSL or not.

Another option concerning port 8000 is to expose both 8000 and 8443 and enable [https_only](https://getkong.org/docs/latest/proxy/#the-https_only-property) which is used to notify clients to upgrade to https from http.
