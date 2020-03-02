# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VCN.

Print the internal IP address and Pod CIDR range for each worker instance:

```
for instance in worker-0 worker-1 worker-2; do
  instance_id=$(oci compute instance list \
    --compartment-id $C --raw-output \
    --query "data[?\"display-name\" == '$instance'] | [?\"lifecycle-state\" == 'RUNNING'] | [0].\"id\"")
  internal_ip=$(oci compute instance list-vnics --instance-id $instance_id --raw-output --query 'data[0]."private-ip"')
  pod_cidr=$(oci compute instance get --instance-id  $instance_id --raw-output --query 'data."extended-metadata"."prod-cidr"')
  echo "$internal_ip $pod_cidr"
done
```

> output

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## Routes

Retrieve the VCN `kubernetes-the-hard-way`:
```
VCN=$(oci network vcn list \
  --compartment-id $C --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-the-hard-way']|[0].id")
```
Retrieve the default routing table:
```
RT=$(oci network route-table list \
  --compartment-id $C --vcn-id $VCN --raw-output \
  --query 'data[0].id')
```

Retrieve the subnet for kubernetes:
```
SUBNET=$(oci network subnet list \
  --compartment-id $C --vcn-id $VCN --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-subnet']|[0].id")
```

Retrieve the Internet Gateway:
```
IGW=$( oci network internet-gateway list \
  --compartment-id $C --vcn-id $VCN --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-igw']|[0].id")
```

Create network routes for each worker instance:

```
routes="{\"destination\": \"0.0.0.0/0\",
        \"destination-type\": \"CIDR_BLOCK\",
        \"network-entity-id\": \"$IGW\"}"

for i in 0 1 2; do
  internal_ip_id=$(oci network private-ip list \
    --ip-address 10.240.0.2$i --subnet-id $SUBNET \
    --raw-output --query "data [?\"ip-address\" == '10.240.0.2$i'] | [0].id")

  routes="{
      \"destination\": \"10.200.$i.0/24\",
      \"destination-type\": \"CIDR_BLOCK\",
      \"network-entity-id\": \"$internal_ip_id\"
    }, $routes"
done

oci network route-table update --rt-id $RT \
  --force \
  --route-rules "[$routes]"
```

List the routes in the `kubernetes-the-hard-way` VCN:

```
oci network route-table get --rt-id $RT --query 'data."route-rules"'
```

> output

```
[
  {
    "cidr-block": null, 
    "description": null, 
    "destination": "10.200.2.0/24", 
    "destination-type": "CIDR_BLOCK", 
    "network-entity-id": "ocid1.privateip.oc1.iad.abuwcljrmubmenmhb7s7a3ixe47h7sv55knmqqq7iaxm4rbxux6sq6cnwbwa"
  }, 
  {
    "cidr-block": null, 
    "description": null, 
    "destination": "10.200.1.0/24", 
    "destination-type": "CIDR_BLOCK", 
    "network-entity-id": "ocid1.privateip.oc1.iad.abuwcljtlqddmqik26rkulvdint46aqcvsnrnnddwpvwf43uxp2nx77yjngq"
  }, 
  {
    "cidr-block": null, 
    "description": null, 
    "destination": "10.200.0.0/24", 
    "destination-type": "CIDR_BLOCK", 
    "network-entity-id": "ocid1.privateip.oc1.iad.abuwcljsev5adv56wv4llnqz3vkm54ahdhlidyblgjhtxhchglgga7dcscwq"
  }, 
  {
    "cidr-block": null, 
    "description": null, 
    "destination": "0.0.0.0/0", 
    "destination-type": "CIDR_BLOCK", 
    "network-entity-id": "ocid1.internetgateway.oc1.iad.aaaaaaaaf6foj3bpe6azbgkuym7yw755qe6uraqkie3a6vammnint7xtg3ia"
  }
]

```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
