# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
for name in worker controller; do
  for i in 0 1 2; do
    instance_id=$(oci compute instance list \
      --compartment-id $C --raw-output \
      --query "data[?\"display-name\" == '${name}-$i'] | [?\"lifecycle-state\" == 'RUNNING'] | [0].\"id\"")
    oci compute instance terminate --force --instance-id $instance_id
  done
done
```

## Networking

Delete the public load balancer network resources:

```
# delete load balancer
LB=$(oci lb load-balancer list --compartment-id $C \
  --raw-output --query "data [?\"display-name\" == 'kubernetes-lb']|[0].\"id\"")
oci lb load-balancer delete --force --load-balancer-id $LB

VCN=$(oci network vcn list \
  --compartment-id $C --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-the-hard-way']|[0].id")

# delete subnet
LB_SUBNET=$(oci network subnet list \
  --compartment-id $C --vcn-id $VCN --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-lb']|[0].id")
oci network subnet delete --wait-for-state TERMINATED --force --subnet-id $LB_SUBNET

# delete security list
LB_SL=$(oci network security-list list --compartment-id  $C \
  --vcn-id $VCN --raw-output \
  --query "data[?\"display-name\" == 'kubernetes-lb'] | [0].\"id\"")
oci network security-list delete --security-list-id $LB_SL
```

Delete the `kubernetes-the-hard-way` VCN resouces:
```
# delete subnet
SUBNET=$(oci network subnet list \
  --compartment-id $C --vcn-id $VCN --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-subnet']|[0].id")
oci network subnet delete --wait-for-state TERMINATED --subnet-id $SUBNET

# delete route rules
RT=$(oci network route-table list --compartment-id $C --vcn-id $VCN --raw-output --query 'data[0].id')
oci network route-table update --force --route-rules "[]" --rt-id=$RT

# delete internet-gateway
IGW=$(oci network internet-gateway list \
  --compartment-id $C --vcn-id $VCN --raw-output \
  --query "data [?\"display-name\" == 'kubernetes-igw']|[0].id")
oci network internet-gateway delete --wait-for-state TERMINATED --ig-id $IGW

# delete vcn
oci network vcn delete --wait-for-state TERMINATED --vcn-id $VCN
```
