# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients and the `admin` user.

> The `scheduler` and `controller manager` access the Kubernetes API Server locally over an insecure API port which does not require authentication. The Kubernetes API Server's insecure port is only enabled for local access.

### Kubernetes Public IP Address

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability an external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the `kubernetes-the-hard-way` load balancer DNS name:

```
KUBERNETES_ENDPOINT=$(aws elbv2 describe-load-balancers \
  --output text \
  --names kubernetes-the-hard-way \
  --query LoadBalancers[*].DNSName)
```

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

Generate a kubeconfig file for each worker node:

```
for instance in worker-0 worker-1 worker-2; do
  PRIVATE_DNS_NAME=$(aws ec2 describe-instances \
    --output text \
    --filters "{\"Name\":\"tag:Name\", \"Values\":[\"${instance}\"]}" "{\"Name\":\"instance-state-name\", \"Values\":[\"running\"]}" \
    --query 'Reservations[*].Instances[*].PrivateDnsName' | cut -d '.' -f1)

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_ENDPOINT}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${PRIVATE_DNS_NAME} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${PRIVATE_DNS_NAME} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

> results:

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_ENDPOINT}:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```

```
kubectl config use-context default \
  --kubeconfig=kube-proxy.kubeconfig
```
Results:
```
kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```
Results:

```
kube-controller-manager.kubeconfig
```

### The kube-scheduler Kubernetes Configuration File
Generate a kubeconfig file for the `kube-scheduler` service:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

Results:

```
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File
Generate a kubeconfig file for the `admin` user:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

Results:

```
admin.kubeconfig
```

## Distribute the Kubernetes Configuration Files

Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

```
for instance in worker-0 worker-1 worker-2; do
  scp -i ~/.ssh/kubernetes-the-hard-way.pem ${instance}.kubeconfig kube-proxy.kubeconfig ubuntu@$(ip_from_instance "$instance"):~/
done
```

Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

```
for instance in controller-0 controller-1 controller-2; do
  scp -i ~/.ssh/kubernetes-the-hard-way.pem admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@$(ip_from_instance "$instance"):~/
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
