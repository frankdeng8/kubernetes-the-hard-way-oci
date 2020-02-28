# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster in a [region](https://docs.cloud.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm) and in a [compartment](https://docs.cloud.oracle.com/en-us/iaas/Content/GSG/Tasks/choosingcompartments.htm).

> Ensure the region has been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-region) lab.

> Ensure the [compartment](https://docs.cloud.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcompartments.htm) has been created and you have the permissions in it.

Retrieve the compartment OCID:
```
C=$(oci iam compartment list --raw-output --query 'data[0].id')
echo $C
```

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Cloud Network

In this section a dedicated [Virtual Cloud Network]() (VCN) will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VCN:

```
VCN=$(oci network vcn create --compartment-id $C \
  --display-name kubernetes-the-hard-way \
  --dns-label thehardway \
  --cidr-block 10.240.0.0/16 \
  --raw-output --query 'data.id')
echo $VCN
```

### Subnet

A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` regional subnet in the `kubernetes-the-hard-way` VCN:

```
SUBNET=$(oci network subnet create --compartment-id $C \
  --vcn-id $VCN \
  --display-name kubernetes-subnet \
  --dns-label kubernetes \
  --cidr-block 10.240.0.0/24 \
  --raw-output --query 'data.id')
echo $SUBNET
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Internet Gateway

Create an internet gateway that allows internet access:
```
IGW=$(oci network internet-gateway create \
  --compartment-id $C \
  --vcn-id $VCN \
  --is-enabled true \
  --display-name kubernetes-igw \
  --raw-output --query 'data.id')
echo $IGW
```

### Routing table

Find the default routing table:
```
RT=$(oci network route-table list --compartment-id $C --vcn-id $VCN --raw-output --query 'data[0].id')
echo $RT
```
Add a routing rule to the default routing table:
```
oci network route-table update \
  --rt-id $RT \
  --force \
  --route-rules "[{\"cidrBlock\": \"0.0.0.0/0\", \"networkEntityId\": \"$IGW\"}]"
```

### Security List

Find the default security list:
```
SL=$(oci network security-list list \
  --compartment-id $C \
  --vcn-id $VCN \
  --raw-output --query 'data[0].id')
echo $SL
```
Add rules to the default security list to allows internal communication across all protocols, and external SSH, ICMP, and HTTPS(kubernetes API server):

```
oci network security-list update \
  --force \
  --security-list-id $SL \
  --egress-security-rules '[{"destination": "0.0.0.0/0", "destinationType": "CIDR_BLOCK", "protocol": "all", "isStateless": false}]' \
  --ingress-security-rules '[{"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 6, "isStateless": false, "tcpOptions": {"destinationPortRange": {"max": 22, "min": 22}}},  {"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 1, "isStateless": false}, {"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 6, "isStateless": false, "tcpOptions": {"destinationPortRange": {"max": 6443, "min": 6443}}}, {"source": "10.240.0.0/24", "sourceType": "CIDR_BLOCK", "protocol": "all", "isStateless": false}, {"source": "10.200.0.0/16", "sourceType": "CIDR_BLOCK", "protocol": "all", "isStateless": false}]'
```

List the default security rules in the `kubernetes-the-hard-way` VCN:

```
oci network security-list get --security-list-id $SL
```

> output

```
{
  "data": {
    "compartment-id": "<compartment OCID>", 
    "defined-tags": {}, 
    "display-name": "Default Security List for kubernetes-the-hard-way", 
    "egress-security-rules": [
      {
        "description": null, 
        "destination": "0.0.0.0/0", 
        "destination-type": "CIDR_BLOCK", 
        "icmp-options": null, 
        "is-stateless": false, 
        "protocol": "all", 
        "tcp-options": null, 
        "udp-options": null
      }
    ], 
    "freeform-tags": {}, 
    "id": "<Security list OCID>", 
    "ingress-security-rules": [
      {
        "description": null, 
        "icmp-options": null, 
        "is-stateless": false, 
        "protocol": "6", 
        "source": "0.0.0.0/0", 
        "source-type": "CIDR_BLOCK", 
        "tcp-options": {
          "destination-port-range": {
            "max": 22, 
            "min": 22
          }, 
          "source-port-range": null
        }, 
        "udp-options": null
      }, 
      {
        "description": null, 
        "icmp-options": null, 
        "is-stateless": false, 
        "protocol": "1", 
        "source": "0.0.0.0/0", 
        "source-type": "CIDR_BLOCK", 
        "tcp-options": null, 
        "udp-options": null
      }, 
      {
        "description": null, 
        "icmp-options": null, 
        "is-stateless": false, 
        "protocol": "6", 
        "source": "0.0.0.0/0", 
        "source-type": "CIDR_BLOCK", 
        "tcp-options": {
          "destination-port-range": {
            "max": 6443, 
            "min": 6443
          }, 
          "source-port-range": null
        }, 
        "udp-options": null
      }, 
      {
        "description": null, 
        "icmp-options": null, 
        "is-stateless": false, 
        "protocol": "all", 
        "source": "10.240.0.0/24", 
        "source-type": "CIDR_BLOCK", 
        "tcp-options": null, 
        "udp-options": null
      }, 
      {
        "description": null, 
        "icmp-options": null, 
        "is-stateless": false, 
        "protocol": "all", 
        "source": "10.200.0.0/16", 
        "source-type": "CIDR_BLOCK", 
        "tcp-options": null, 
        "udp-options": null
      }
    ], 
    "lifecycle-state": "AVAILABLE", 
    "time-created": "2020-02-25T07:36:12.815000+00:00", 
    "vcn-id": "<VCN OCID>"
  }, 
  "etag": "17f9c750"
}

```

## Compute Instances

The compute instances in this lab will be provisioned using [Oracle Linux](https://www.oracle.com/linux/) 7, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address and a fixed public IP address to simplify the Kubernetes bootstrapping process.

List and choose latest Oracle Linux 7 image:
```
IMG=$(oci compute image list \
  --compartment-id $C \
  --operating-system "Oracle Linux" \
  --operating-system-version 7.7 --sort-by TIMECREATED \
   --raw-output \
  --query "data [?starts_with(\"display-name\", 'Oracle-Linux-7.7-20')]|[0].\"id\"")
echo $IMG
```

List and choose the instance shape:
```
oci compute shape list --compartment-id $C --output table
```

List and choose the availability domain:
```
AD=$(oci iam availability-domain list --raw-output --query 'data[0].name')
echo $AD
AD_PREFIX=${AD%-?}
echo $AD_PREFIX
```
Create an [SSH key pair](https://docs.cloud.oracle.com/en-us/iaas/Content/Compute/Tasks/managingkeypairs.htm) to connect to instances:
```
ssh-keygen -t rsa -N "" -b 4096
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane in three Availability Domains:

```
for i in 0 1 2; do
  oci compute instance launch \
    --availability-domain ${AD_PREFIX}-$((i+1)) \
    --compartment-id $C \
    --display-name controller-${i} \
    --freeform-tags '{"kubernetes": "controller"}' \
    --image-id $IMG \
    --shape VM.Standard.E2.1 \
    --subnet-id $SUBNET \
    --private-ip 10.240.0.1${i} \
    --assign-public-ip true \
    --ssh-authorized-keys-file ~/.ssh/id_rsa.pub
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  oci compute instance launch \
    --compartment-id $C \
    --availability-domain ${AD_PREFIX}-$((i+1)) \
    --display-name worker-${i} \
    --freeform-tags '{"kubernetes": "worker"}' \
    --image-id $IMG \
    --shape VM.Standard.E2.1 \
    --subnet-id $SUBNET \
    --private-ip 10.240.0.2${i} \
    --assign-public-ip true \
    --skip-source-dest-check true \
    --ssh-authorized-keys-file ~/.ssh/id_rsa.pub
done
```

### Verification

List the compute instances in your default region and specified compartment:

```
oci compute instance list --compartment-id $C \
  --output table \
  --query 'data [].{NAME:"display-name", AD: "availability-domain", SHAPE: "shape", STATUS: "lifecycle-state"}'
```

> output

```
+----------------------+--------------+------------------+------------+
| AD                   | NAME         | SHAPE            | STATUS     |
+----------------------+--------------+------------------+------------+
| nzlf:US-ASHBURN-AD-1 | controller-0 | VM.Standard.E2.1 | RUNNING    |
| nzlf:US-ASHBURN-AD-1 | worker-0     | VM.Standard.E2.1 | RUNNING    |
| nzlf:US-ASHBURN-AD-2 | controller-1 | VM.Standard.E2.1 | RUNNING    |
| nzlf:US-ASHBURN-AD-2 | worker-1     | VM.Standard.E2.1 | RUNNING    |
| nzlf:US-ASHBURN-AD-3 | controller-2 | VM.Standard.E2.1 | RUNNING    |
| nzlf:US-ASHBURN-AD-3 | worker-2     | VM.Standard.E2.1 | RUNNING    |
+----------------------+--------------+------------------+------------+
```

### Verifying SSH Access


Log into the `controller-0` instance:
```
ssh opc@<controller-0 public IP>
```

## Kubernetes Public IP Address from Public Load balancer

Create a public load balancer fronting the Kubernetes API Servers.
The public IP address of the load balancer will be the public IP address for Kubernetes.

### Security list

Create a security list for load balancer in `Kubernetes-the-hard-way` VCN, allow ingress traffic for Kubernetes API server port 6443 and all egress traffic.

```
LB_SE=$(oci network security-list create \
  --display-name kubernetes-lb \
  --compartment-id $C \
  --vcn-id $VCN \
  --egress-security-rules '[{"destination": "0.0.0.0/0", "destinationType": "CIDR_BLOCK", "protocol": "all", "isStateless": false}]' \
  --ingress-security-rules '[{"source": "0.0.0.0/0", "sourceType": "CIDR_BLOCK", "protocol": 6, "isStateless": false, "tcpOptions": {"destinationPortRange": {"max": 6443, "min": 6443}}} ]' \
  --raw-output --query 'data.id')
echo $LB_SE
```
### Subnet
Create the `kubernetes-lb` regional subnet for the load balancer in the `kubernetes-the-hard-way` VCN:

```
LB_SUBNET=$(oci network subnet create --compartment-id $C \
  --vcn-id $VCN \
  --display-name kubernetes-lb \
  --dns-label kuberneteslb \
  --security-list-ids "[\"$LB_SE\"]" \
  --cidr-block 10.240.10.0/24 \
  --raw-output --query 'data.id')
echo $LB_SUBNET
```
### Public Load Balancer
Create a Public Load Balancer:

```
oci lb load-balancer create --compartment-id $C \
  --display-name kubernetes-lb \
  --is-private false \
  --shape-name 100Mbps \
  --subnet-ids "[\"$LB_SUBNET\"]"
```
Retrieve the Load Balancer ID:
```
LB=$(oci lb load-balancer list --compartment-id $C \
  --raw-output --query "data [?\"display-name\" == 'kubernetes-lb']|[0].\"id\"")
echo $LB
```

List the Load Balancer:
```
oci lb load-balancer get --load-balancer-id $LB
```

Retrieve the public IP address of the load balancer fronting the Kubernetes API Servers:
```
KUBERNETES_PUBLIC_ADDRESS=$(oci lb load-balancer get --load-balancer-id $LB \
  --raw-output --query 'data."ip-addresses"|[0]."ip-address"')
echo $KUBERNETES_PUBLIC_ADDRESS
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
