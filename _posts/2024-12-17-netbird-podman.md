---
layout: post
title: Netbird podman
published: false
---

How to migrate Netbird quickstart install to podman
<!--more-->

## Intro

Netbird is a very cool project that makes it easy to set-up a private network, in case for example you need to access some private servers from your work environment, or for sharing with family.

Netbird is open source friendly and they provide a set-up script that let you set-up netbird on a self-hosted infrastructure, so that you can play with it and eventually convince your hierarchy this should be be used in your company. At which stage you could use the cloud version or get support. Or simply continue with the free self-hosted solution.

The set-up script is very nice, in five minutes it gives you a running netbird. However it runs in docker compose, and companies like mine (red-hat based) prefer to use podman. Luckily migrating from docker compose to podman is easy, especially now that red-hat has made it so easy to run podman containers via systemd (quadlet).

## How to migrate to podman

In order to use podman you will have to remove docker from the system and install podman. I show it for red-hat based distro (in this example I use a rocky linux on GCP).

```
dnf remove docker
dnf install podman podman-tools
```

```bash
minikube start --driver podman --cpus 8 --memory 8G --disk-size 10G
```

Install grafana loki in single binary mode (only one replica)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm update
```

to see the available charts
```bash
helm search repo
```

create file loki-single-values.yaml

```yaml
loki:
  commonConfig:
    replication_factor: 1
  storage:
    type: 'filesystem'
singleBinary:
  replicas: 1
```

create a namespace grafana: we will install all our pods/deployment in this namespace

```ssh
kubectl create namespace grafana
helm install loki --values loki-single-values.yaml --namespace=grafana grafana/loki
```

After a few minutes all the pods should be created
```ssh
kubectl get pods -n grafana
# you can get inside a pod to see what it looks like (config for example)
kubectl -n grafana exec -ti loki-0 /bin/sh
# to see the logs
kubectl -n grafana logs loki-0 -f
# decribe loki pod
kubectl describe pod/loki-0 -n grafana
```

Now we need to deploy grafana in the same namespace

```
helm install --namespace=grafana --set persistence.enabled=true grafana grafana/grafana
```

```text
[pierre@fedora single]$ helm install --namespace=grafana --set persistence.enabled=true grafana grafana/grafana
NAME: grafana
LAST DEPLOYED: Fri Jul 21 19:12:06 2023
NAMESPACE: grafana
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.grafana.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace grafana -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace grafana port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
```

Follow the instructions above to log in to grafana, in my case the pod name is grafana-bbc7b96c5-clwd4, so I run

```ssh
kubectl --namespace grafana port-forward grafana-bbc7b96c5-clwd4 3000
```

and then I can connect on my laptop on http://localhost:3000 with user admin

Create a Loki datasource : go to Home->Connections->Add new Connection. In the URL specify http://loki:3100. Add a Custom HTTP Header: X-Scope-OrgId = mytenant


Let us add a pod with promtail in then cluster. For that we must build an image and upload it on the minikube repository

```bash
minikube ssh
```
inside the minikube we can build the image based on this Dockerfile. Note that in order to have systemd in the container we use ubi8-init

```text
FROM registry.access.redhat.com/ubi8-init
RUN dnf install -y https://github.com/grafana/loki/releases/download/v2.8.2/promtail-2.8.2.x86_64.rpm
RUN dnf install -y https://github.com/grafana/loki/releases/download/v2.8.2/logcli-2.8.2.x86_64.rpm
```

```bash
docker build -t mypromtail .
```

Prepare a persistent volume for the promtail container (so that we can safe our config)

```bash
minikube ssh
mkdir /data/pv001
exit
```

create a file pv001.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
  labels:
    type: local
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1M
  hostPath:
    path: "/data/pv001"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv001-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1M
```

```bash
kubectl apply -n grafana -f pv001.yaml
```

create a pod containing promtail. The privileged flag is needed for systemd

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: promtail
  labels:
    app: promtail
spec:
  volumes:
    - name: promtail-data
      persistentVolumeClaim:
        claimName: pv001-claim
  containers:
    - name: promtail
      imagePullPolicy: Never
      image: mypromtail
      securityContext:
        privileged: true
      volumeMounts:
        - mountPath: /config
          name: promtail-data
```

```bash
kubectl apply -f promtail.yaml -n grafana
```

Now we can get into the promtail pod, edit the promtail config, restart promtail and check the data via grafana


