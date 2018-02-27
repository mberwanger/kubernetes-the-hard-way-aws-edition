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
ssh-aws controller-0 \
  ETCDCTL_API=3 etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a e8 bb 39 30 e5 c2 25  |:v1:key1:..90..%|
00000050  39 3a a2 01 30 b0 01 09  b3 75 de 8f 8d 4a de b9  |9:..0....u...J..|
00000060  b4 24 6a 37 2a 35 d0 46  6c fc c4 c7 2d ad c0 3f  |.$j7*5.Fl...-..?|
00000070  5c 16 01 2e dc ac cb 40  60 62 4b 2d ff e8 9d 0c  |\......@`bK-....|
00000080  db f2 20 b6 79 79 80 97  11 09 2d 0e fb b0 42 af  |.. .yy....-...B.|
00000090  21 f2 cd 94 88 72 65 fb  ca 1e 61 21 3d 99 4a 45  |!....re...a!=.JE|
000000a0  67 7c fd 8c f2 61 91 7a  c3 9b 1d b4 37 15 0d ef  |g|...a.z....7...|
000000b0  cf 23 5b 63 df 43 f4 63  0a 4a 95 2f eb d2 9d 3b  |.#[c.C.c.J./...;|
000000c0  ce d2 f4 d1 f6 3d de e1  6f 1c 48 31 68 24 ca 5b  |.....=..o.H1h$.[|
000000d0  87 ca a3 4b f6 1d 91 2d  52 42 4c 77 0e 10 d7 ba  |...K...-RBLw....|
000000e0  4d d4 4e 88 64 bd a5 4d  91 0a                    |M.N.d..M..|
000000ea
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl run nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l run=nginx
```

> output

```
NAME                     READY     STATUS    RESTARTS   AGE
nginx-4217019353-b5gzn   1/1       Running   0          15s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.13.7
Date: Mon, 18 Dec 2017 14:50:36 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 Nov 2017 14:28:04 GMT
Connection: keep-alive
ETag: "5a1437f4-264"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
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
127.0.0.1 - - [26/Feb/2018:21:23:09 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.54.0" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.13.9
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

Create a security group rule that allows remote access to the `nginx` node port:

```
VPC_ID=$(aws ec2 describe-vpcs \
  --output text \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" \
  --query 'Vpcs[*].VpcId')
```

```
NGINX_SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name kubernetes-the-hard-way-allow-nginx-service \
  --description "Allows external access to nginx service" \
  --vpc-id ${VPC_ID} | jq -r '.GroupId')
```

```
aws ec2 authorize-security-group-ingress \
  --group-id ${NGINX_SECURITY_GROUP_ID} \
  --ip-permissions "[ {\"IpProtocol\": \"tcp\", \"FromPort\": ${NODE_PORT}, \"ToPort\": ${NODE_PORT}, \"IpRanges\": [{\"CidrIp\": \"0.0.0.0/0\"}]} ]"
```

```
for i in 0 1 2; do
  INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=worker-${i}" | jq -r '.Reservations[].Instances[].InstanceId')

  EXISTING_SECURITY_GROUPS=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=worker-${i}" | jq -r '.Reservations[].Instances[].SecurityGroups[].GroupId' | tr '\n' ' ')

  aws ec2 modify-instance-attribute \
    --instance-id ${INSTANCE_ID} \
    --groups ${EXISTING_SECURITY_GROUPS} ${NGINX_SECURITY_GROUP_ID}
done
```

Retrieve the external IP address of a worker instance:

```
PUBLIC_ADDRESS=$(aws ec2 describe-instances \
  --output text \
  --filters "Name=tag:Name,Values=worker-0" \
  --query 'Reservations[*].Instances[*].PublicIpAddress')  
```

Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${PUBLIC_ADDRESS}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.13.9
Date: Mon, 26 Feb 2018 21:24:45 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 20 Feb 2018 12:21:20 GMT
Connection: keep-alive
ETag: "5a8c12c0-264"
Accept-Ranges: bytes
```

Next: [Cleaning Up](14-cleanup.md)
