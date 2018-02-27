# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

## Routes

Create network routes for each worker instance:

```
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" | \
  jq -r '.RouteTables[].RouteTableId')
```

```
for i in 0 1 2; do
  INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=worker-${i}" | jq -j '.Reservations[].Instances[].InstanceId')

  aws ec2 create-route \
    --route-table-id ${ROUTE_TABLE_ID} \
    --destination-cidr-block 10.200.${i}.0/24 \
    --instance-id ${INSTANCE_ID}  
done
```

List the routes in the `kubernetes-the-hard-way` VPC network:

```
aws ec2 describe-route-tables \
  --output table \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" \
  --query "RouteTables[*].{Routes:Routes[*].{InstanceOwnerId:InstanceOwnerId,DestinationCidrBlock:DestinationCidrBlock,InstanceId:InstanceId,State:State,NetworkInterfaceId:NetworkInterfaceId}}"
```

> output

```
------------------------------------------------------------------------------------------------------
|                                         DescribeRouteTables                                        |
||                                              Routes                                              ||
|+----------------------+----------------------+------------------+----------------------+----------+|
|| DestinationCidrBlock |     InstanceId       | InstanceOwnerId  | NetworkInterfaceId   |  State   ||
|+----------------------+----------------------+------------------+----------------------+----------+|
||  10.200.0.0/24       |  i-030daf3bc83135224 |  896174747632    |  eni-3ad5aef6        |  active  ||
||  10.200.1.0/24       |  i-01a4616a119011ad5 |  896174747632    |  eni-1bd6add7        |  active  ||
||  10.200.2.0/24       |  i-0a1f305b7e4851a8d |  896174747632    |  eni-4dcbb081        |  active  ||
||  10.240.0.0/16       |  None                |  None            |  None                |  active  ||
||  0.0.0.0/0           |  None                |  None            |  None                |  active  ||
|+----------------------+----------------------+------------------+----------------------+----------+|
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
