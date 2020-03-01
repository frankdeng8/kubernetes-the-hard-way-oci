# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
instance_id=$(oci compute instance list \
  --compartment-id $C --raw-output \
  --query "data[?\"display-name\" == 'controller-0'] | [?\"lifecycle-state\" == 'RUNNING'] | [0].\"id\"")

external_ip=$(oci compute instance list-vnics --instance-id $instance_id --raw-output --query 'data[0]."public-ip"')

ssh opc@$external_ip \
  "sudo ETCDCTL_API=3 /usr/local/bin/etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 3a 66 37 cc c0 8f da  |:v1:key1::f7....|
00000050  e0 b2 a2 f5 b0 ab dd 48  90 8d fd d1 2a aa ac 5d  |.......H....*..]|
00000060  bd 70 75 67 e1 10 0b 18  f1 0e 6d 59 b8 1f 17 f6  |.pug......mY....|
00000070  fa 80 ee 25 b7 7e b8 8e  75 d3 6c f4 46 5c 2e 35  |...%.~..u.l.F\.5|
00000080  db 43 f6 7d b6 1c 0e 3d  78 5f 7f b7 8c 6e 62 53  |.C.}...=x_...nbS|
00000090  a4 68 17 f8 c4 eb 93 db  6a 49 4a d6 34 4e 9d 4e  |.h......jIJ.4N.N|
000000a0  97 0c ba 64 b2 7f 40 76  0e e5 5c 08 4d 53 6b d5  |...d..@v..\.MSk.|
000000b0  5f 73 c3 3a 59 b0 14 ca  e8 c9 5c fb dc f2 82 74  |_s.:Y.....\....t|
000000c0  89 71 7d bc dd 11 11 de  6f d6 c4 7c cb 5c c1 ca  |.q}.....o..|.\..|
000000d0  82 4a fc 2b db 08 e5 8a  33 62 3d 7c 8e 56 91 48  |.J.+....3b=|.V.H|
000000e0  54 c7 c0 1a 2a 27 83 39  42 0a                    |T...*'.9B.|
000000ea
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l app=nginx
```

> output

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-554b9c67f9-vt5rn   1/1     Running   0          10s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.8
Date: Sun, 01 Mar 2020 20:31:06 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 Jan 2020 13:36:08 GMT
Connection: keep-alive
ETag: "5e26fe48-264"
Accept-Ranges: bytes

```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
127.0.0.1 - - [01/Mar/2020:20:31:06 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.17.8
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Retrieve the VCN `kubernetes-the-hard-way`:
```
VCN=$(oci network vcn list \
  --compartment-id $C --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-the-hard-way']|[0].id")
```

Retrieve the default security list:

```
SL=$(oci network security-list list --compartment-id  $C \
  --vcn-id $VCN --raw-output \
  --query "data[?contains(\"display-name\", 'Default')] | [0].\"id\"")
```

Update the security rule that allows remote access to the default node ports range 30000-32767:
```
oci network security-list update \
  --force \
  --security-list-id $SL \
  --ingress-security-rules '[{"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 6, "isStateless": false, "tcpOptions": {"destinationPortRange": {"max": 22, "min": 22}}},  {"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 1, "isStateless": false}, {"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 6, "isStateless": false, "tcpOptions": {"destinationPortRange": {"max": 6443, "min": 6443}}}, {"source": "10.240.0.0/24", "sourceType": "CIDR_BLOCK", "protocol": "all", "isStateless": false}, {"source": "10.200.0.0/16", "sourceType": "CIDR_BLOCK", "protocol": "all", "isStateless": false}, {"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 6, "isStateless": false, "tcpOptions": {"destinationPortRange": {"max": 32767, "min": 30000}}}]'
```

Retrieve the external IP address of a worker instance:

```
instance_id=$(oci compute instance list \
  --compartment-id $C --raw-output \
  --query "data[?\"display-name\" == 'worker-0'] | [?\"lifecycle-state\" == 'RUNNING'] | [0].\"id\"")

EXTERNAL_IP=$(oci compute instance list-vnics --instance-id $instance_id --raw-output --query 'data[0]."public-ip"')
```

Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Sat, 14 Sep 2019 21:12:35 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 08:50:00 GMT
Connection: keep-alive
ETag: "5d5279b8-264"
Accept-Ranges: bytes
```

Next: [Cleaning Up](14-cleanup.md)
