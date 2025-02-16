---
layout: post
title: Netbird troubleshooting
published: false
---

How to troubleshoot P2P connections with Netbird
<!--more-->

## Intro

Netbird is a very cool project that makes it easy to set-up a private network. I am lucky enough to spend time learning it because my company is evaluating using Netbird instead of Fortinet as a VPN solution.

Netbird is open source friendly and they provide a set-up script that lets you set-up netbird on a self-hosted infrastructure very quickly, so that you can play with it. Later you can decide to use the cloud version or continue using self-hosted but with support and enterprise features. Or simply continue with the free self-hosted solution.

At the core of a Netbird network is Wireguard: Netbird uses wireguard tunnel to have peers in the network communicating to each other. Another key technology used by Netbird to make the peer to peer connection possible is Webrtc. Webrtc will enable two peers (each peer being potentially behind nat devices and firewall) to establish a direct connection, and in case it is impossible (unfriendly nat, that cannot be traversed) webrtc will automatically falls back to a relay mechanism (using an external server, called a TURN server).

Because direct peer to peer connections are better than relayed connections, my goal was to make sure the routing peers inside our private cloud would be reachable directly without relay. But for some reason it was not working. To troubleshoot the issue, and before asking help to the internal network experts, I decided to study the technology, in particular webrtc because it is at the heart of the connection issue. This turned out to be a super learning opportunity, because webrtc involves STUN/TURN (traversal utilities for NAT) which in turns requires a good understanding of NAT. And since Netbird is written in Go, they use a popular library called PION. PION is an implementation in Go of the webrtc protocol (webrtc is originally a javascript in the browser framework). Once I knew enough about webrtc and PION to do my own small test lab, I could see webrtc was not the issue and there was something else in conjunction with netbird causing the issue. Guided by a network export, I had to understand in a more detailed way how the linux kernel is routing packets and how netbird interact with that...  This piece of software looks really amazing and by understanding it one can learn a lot about golang and networking. Not to mention that this knowledge is very useful in order to maintain a self-hosted netbird.

## How to learn and play with webrtc


with webrtc two peers that want to communicate with each other will use a out-of-band communication mechanism (i.e. not part of the webrtc protocol) to exchange connection candidates (basically pairs of ip address and udp port). This out of band mechanism is usually a signaling server. So to test webrtc you first need to create a signaling server. Developing a signaling server in golang for a basic testing peer to peer setup is not very difficult, there are plenty of resources. The more sensible choice would be to do a websocket but to keep it to the simplest possible I did it with plain http (and with the help of ChatGPT). Even though the code is not very long nor involved, there are valuable techniques in golang that I learned. Especially the way a kind of long polling is made, meaning a peer will send a get request to the signaling web server, and the signaling server will "block" the request, pulling messages from a channel and sending them to the client as they arrive, without stopping the request. This is very golang idiomatic.

Note: I will certainly re-implement the signaling part using websocket, as it is probably trivial to do, but for now this is largely sufficient for my setup.


There are a lot of good resources on webrtc, here are my favorite



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


