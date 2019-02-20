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

# Configuring OpenFaas Endpoints inside Kong
You can do this task with two ways, command line or through KONGA web interface.

Here you can get with command line to configure endpoits.

