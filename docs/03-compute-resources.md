# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster in a [region](https://docs.cloud.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm) and in a [compartment](https://docs.cloud.oracle.com/en-us/iaas/Content/GSG/Tasks/choosingcompartments.htm).

> Ensure the region has been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-region) lab.

> Ensure the [compartment](https://docs.cloud.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcompartments.htm) has been created and you have the permissions in it.

Get the compartment OCID:
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
V=$(oci network vcn create --compartment-id $C \
  --display-name kubernetes-the-hard-way \
  --dns-label thehardway \
  --cidr-block 10.240.0.0/24 \
  --raw-output --query 'data.id')
echo $V
```

A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```
S=$(oci network subnet create --compartment-id $C \
  --vcn-id $V \
  --display-name kubernetes \
  --cidr-block 10.240.0.0/24 \
  --raw-output --query 'data.id')
echo $S
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

Create an internet gateway that allows internet access:
```
I=$(oci network internet-gateway create \
  --compartment-id $C \
  --vcn-id $V \
  --is-enabled true \
  --display-name kubernetes-the-hard-way-internet-gw \
  --raw-output --query 'data.id')
echo $I
```

Get the default routing table:
```
R=$(oci network route-table list --compartment-id $C --vcn-id $V --raw-output --query 'data[0].id')
echo $R
```
Add a routing rule to the default routing table:
```
oci network route-table update \
  --rt-id $R \
  --force \
  --route-rules "[{\"cidrBlock\": \"0.0.0.0/0\", \"networkEntityId\": \"$I\"}]"
```

### Security List

Get the default security list:
```
SL=$(oci network security-list list \
  --compartment-id $C \
  --vcn-id $V \
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

List the security rules in the `kubernetes-the-hard-way` VPC network:

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

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
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
```
Create an [SSH key pair](https://docs.cloud.oracle.com/en-us/iaas/Content/Compute/Tasks/managingkeypairs.htm) to connect to instances:
```
ssh-keygen -t rsa -N "" -b 4096
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  oci compute instance launch \
    --availability-domain $AD \
    --compartment-id $C \
    --display-name controller-${i} \
    --freeform-tags '{"kubernetes": "controller"}' \
    --image-id $IMG \
    --shape VM.Standard.E2.1 \
    --subnet-id $S \
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
    --availability-domain $AD \
    --compartment-id $C \
    --display-name worker-${i} \
    --freeform-tags '{"kubernetes": "worker"}' \
    --image-id $IMG \
    --shape VM.Standard.E2.1 \
    --subnet-id $S \
    --private-ip 10.240.0.2${i} \
    --assign-public-ip true \
    --ssh-authorized-keys-file ~/.ssh/id_rsa.pub
  }
done
```

### Verification

List the compute instances in your default region and specified compartment:

```
oci compute instance list --compartment-id $C --output table --query 'data [].{NAME:"display-name", AD: "availability-domain", SHAPE: "Shape", STATUS: "lifecycle-state"}'
```

> output

```
+----------------------+--------------+-------+------------+
| AD                   | NAME         | SHAPE | STATUS     |
+----------------------+--------------+-------+------------+
| nzlf:US-ASHBURN-AD-1 | controller-0 | None  | RUNNING    |
| nzlf:US-ASHBURN-AD-1 | controller-1 | None  | RUNNING    |
| nzlf:US-ASHBURN-AD-1 | controller-2 | None  | RUNNING    |
| nzlf:US-ASHBURN-AD-1 | worker-0     | None  | RUNNING    |
| nzlf:US-ASHBURN-AD-1 | worker-1     | None  | RUNNING    |
| nzlf:US-ASHBURN-AD-1 | worker-2     | None  | RUNNING    |
+----------------------+--------------+-------+------------+
```

## Verifying SSH Access


Log into the `controller-0` instance:
```
ssh opc@<controller-0 publich IP>
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
