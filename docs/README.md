Note 2020-07-28T10.48.21
========================
# Network Composure
The purpose of the management network is to provide a way for you, the client, to access and manage the private network where the cluster will reside and be controlled. Think of this as a mantrap, or a DMZ, or even a "Canal Lock" where either a hardware tunnel or client VPN access could be hosted to provide access to private resources within the cluster, and the cluster infrastructure itself.

## Management VPC

Build a management VPC. In GovCloud, this could be done with the VPC wizard that allows you to create one VPC with a public and private subnet.
Make sure to allocate or increase the amount of EIPs you have in your region to accommodate the public address you will need to provision for the VPN gateway.
Once you have your management VPC, proceed to build your VPN hardware tunnel or client access gateway or server.

## Management VPN

After you have a single VPC with public+private subnets, you will want to deploy a VPN server to the public subnet so outside clients can connect to your tenancy. It is recommended to use the OpenVPN Access Server in free mode, using the 2 inbuilt connection licences to start. You could use the open source community edition later on if you don't need the self-service login portal or administrative control panel for configuring your VPN gateway.

Deploy a RHEL 7 server to the management VPC public subnet, create an SSH keypair, and SSH into the server with the public EIP you associate with it. Proceed to install/deploy OpenVPN AS.

You will most likely need to create a public DNS record of your choosing to provide a legitimate hostname for the access server to bind to and use.
Once you login to the AS admin panel, change the binding hostname in the config panel and save it, allowing the VPN server to self-reboot and reapply settings.

For client configurations, you will want to setup DNS settings in the config panel to use the same DNS servers the OpenVPN Access Server uses to resolve names.

## Create Cluster VPC

Once you have figured out the management layer and boundary access component, you can start building the VPC for your cluster to reside in.

Build a VPC with **(3) Private subnets, no NAT or IGW endpoints (internet gateway and NAT)** since this new VPC will not have access to the internet and should be considered an offline or airgapped network. ensure each subnet has to following naming scheme:

```
# Example:
gov-cluster-private-us-gov-west-1a

<name of vpc>-private-<region>-<availability zone>
```

so if you want to name your cluster "mycluster" it would be the following for subnet naming standards:
```
# Example:
mycluster-private-us-gov-east-1a
mycluster-private-us-gov-east-1b
mycluster-private-us-gov-east-1c
```

This detail is important to get right, as it is particular in order for platform discovery on behalf of the installation process to work. Hardcoded stuff!

## Create Transit Gateway

Now you should have two VPCs, one Management and one Cluster VPC. You will need to create a VPC transit gateway, associate both VPCs to the transit gateway, and then update their VPC+Subnet routing tables to enable traffic to flow from the Management VPC to the Cluster VPC. To test, deploy a server to the Cluster VPC in one of the subnets and try to ping it while you are VPN'd into the management network.

If pings fail, try the following:

1. Check NACLs
2. Check Security Groups of the server you're trying to ping and add a `0.0.0.0/0` rule allowing ICMP traffic
3. Check subnet range added to the route tables is correct so traffic can route back and forth between both VPCs.

Note, it may take time for the transit gateway to complete association of the two VPCs. It's basically a soft router as a service that connects the two private network spaces as a hub and spoke to the transit gateway.

## Create DNS Private Hosted Zone (Route53)

Now that the core network bones are in place, you will need to compose your desired DNS zone and base records.

Create a new Private Hosted Zone, using a single subdomain to represent nodes supporting the cluster.

Example:
```
gov-cluster.rht-spo.com
mycluster.mydomain.tld
ocp.my-domain.com
```

Records created under this zone will be much like:
```
registry.mycluster.mydomain.tld
registry.ocp.my-domain.com
registry.gov-cluster.rht-spo.com

# or

api-int.ocp.my-domain.com
```

## Upload RHCOS AMI

1. Create an S3 Bucket named 'rhcos-image', upload your RHCOS image to it
```
aws s3api create-bucket --acl private --bucket rhcos-image --region $AWS_REGION --create-bucket-configuration LocationConstraint=$AWS_REGION
aws s3 cp rhcos*.vmdk s3://rhcos-image
```

2. Create a VMImport role and policy with the following JSON templates in the AWS IAM console:
 Note, the following policy and role allow the vmie service to ingest a disk image as a snapshot to EC2. You have to add this to your account.

**Commands**
```
aws iam create-role --role-name vmimport --assume-role-policy-document "file://templates/vmimport-role.json"
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://templates/vmimport-policy.json"
```

**Role (vmimport-role.json)**
```
{
    "Version": "2012-10-17",
    "Statement": [
       {
          "Effect": "Allow",
          "Principal": { "Service": "vmie.amazonaws.com" },
          "Action": "sts:AssumeRole",
          "Condition": {
             "StringEquals":{
                "sts:Externalid": "vmimport"
             }
          }
       }
    ]
 }
```

**Policy (vmimport-policy.json)**
```
{
    "Version":"2012-10-17",
    "Statement":[
       {
          "Effect": "Allow",
          "Action": [
             "s3:GetBucketLocation",
             "s3:GetObject",
             "s3:ListBucket" 
          ],
          "Resource": [
             "arn:aws-us-gov:s3:::rhcos-image",
             "arn:aws-us-gov:s3:::rhcos-image/*"
          ]
       },
       {
          "Effect": "Allow",
          "Action": [
             "s3:GetBucketLocation",
             "s3:GetObject",
             "s3:ListBucket",
             "s3:PutObject",
             "s3:GetBucketAcl"
          ],
          "Resource": [
             "arn:aws-us-gov:s3:::rhcos-image",
             "arn:aws-us-gov:s3:::rhcos-image/*"
          ]
       },
       {
          "Effect": "Allow",
          "Action": [
             "ec2:ModifySnapshotAttribute",
             "ec2:CopySnapshot",
             "ec2:RegisterImage",
             "ec2:Describe*"
          ],
          "Resource": "*"
       }
    ]
 }
```

3. Import the raw object from S3 as an image snapshot to be usable by EC2:

```
IMAGENAME=$(ls rhcos*)
aws ec2 import-snapshot --region $AWS_REGION --description "Red Hat CoreOS 4 AWS VMDK" --disk-container "Format="VMDK",UserBucket={S3Bucket="rhcos-image",S3Key="$IMAGENAME"}"
```

4.  Wait for the snapshot upload to complete. (requires jq to be installed to work):

```
until [ $IMPORTSTATUS == 'completed' ]; do
    echo $(aws ec2 describe-import-snapshot-tasks | jq '.ImportSnapshotTasks[] .SnapshotTaskDetail.Status' -r)
    IMPORTSTATUS=$(aws ec2 describe-import-snapshot-tasks | jq '.ImportSnapshotTasks[] .SnapshotTaskDetail.Status' -r)
    sleep 5
done
```

5. Grab the snapshot ID, and use it to register a valid AMI:

```
SNAPSHOT_ID=$(aws ec2 describe-import-snapshot-tasks | jq '.ImportSnapshotTasks[] .SnapshotTaskDetail.SnapshotId' -r)
aws ec2 register-image \
   --region $AWS_REGION \
   --architecture x86_64 \
   --description "RHCOS 4" \
   --ena-support \
   --name "RHCOS 4" \
   --virtualization-type hvm \
   --root-device-name '/dev/sda1' \
   --block-device-mappings "DeviceName=/dev/sda1,Ebs={DeleteOnTermination=true,SnapshotId=$SNAPSHOT_ID}"
```

6. Once you register the AMI, you can grab the AMI ID like so:

```
echo -e "\033[1;36mYour Image ID is:" $(aws ec2 describe-images --region us-gov-west-1 --owners $(aws sts get-caller-identity | jq '.Account' -r) | jq '.Images[] .ImageId' -r)
```

**Save your AMI ID for later. You will use it during the formatting and installation of Openshift.**


# Generate and Collate Offline Materials

## Mirror the image content to local directory/media (DATA)
Make sure you use matching `oc` and `openshift-install` binary versions as there is undocumented transient behavior with `oc` not pinning the correct sha256 hash to the mirrored images even when specifying the version of Openshift you want to mirror.

So if you want Openshift 4.5.4, use the 4.5.4 `oc` client. Do not try to use different versions of clients and binaries!

Get your pull secret from https://cloud.redhat.com/openshift/ choosing "create a cluster -> AWS -> IPI" and binaries from there as well.

Then run the following command in order to pull down and create a local offline copy of the images needed for Openshift 4:
```
oc adm release mirror -a ../pull-secret.json --to-dir=mirror quay.io/openshift-release-dev/ocp-release:4.5.4-x86_64
```

This should produce a folder called 'mirror' in your working directory with the blobs and manifests needed to produce your edge registry later on.

## Create certs/keys for TLS, trust, and validation (CERTS)
You will need to generate a single certificate pair in order to provide HTTPS/TLS validated docker registry functionality. I recommend doing this on your workstation that has tools to do this, as RHCOS does not have these tools installed most of the time.
```
openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 365 -out domain.crt
```

During this process, it's going to ask you to fill out a subject name. Make sure you pick a good one, as you'll need to create a matching DNS record later!

Examples:
```
registry.mycluster.mydomain.tld
```

you should have a `domain.crt` and `domain.key`. Make a copy of the .crt file and put it aside, as you will need this for the install-config.yml as well to perform your install later on in order to inject a trust bundle into the manifest generation step.

## Create htpasswd file for authentication purposes (AUTH)
Now you will need to create an htpasswd file in order to mount to the docker registry later on to provide basic user/pass authentication.
```
htpasswd -bBc ./htpasswd <username> <password>
```

Save the username and password for later. you're going to login with podman using this credential later.

## Pull and Export utility images

Pull down and export the following images:

- NGINX
- Docker-Registry

using the following:

```
podman pull docker.io/library/registry:latest
podman pull nginx

# Save the nginx container image to local medium
podman save --quiet -o nginx.tar $(podman images -q nginx:latest)

# Save docker registry container image to local medium
podman save --quiet -o registry.tar $(podman images -q registry:latest)
```

# Edge Registry Composure

## Deploy Node

Deploy a single node named 'registry' or similar to your cluster VPC using the RHCOS AMI you uploaded to your tenancy. For the user data, make sure you use the basic JSON string to inject your authorized key using your SSH key of choice to the node:

```
{"ignition":{"config":{},"security":{"tls":{}},"timeouts":{},"version":"2.2.0"},"networkd":{},"passwd":{"users":[{"name":"core","sshAuthorizedKeys":["YOUR PUBLIC KEY HERE"]}]},"storage":{},"systemd":{}}
```

You can copy/paste the above after replacing the key with your own directly into the "User Data" field when deploying EC2 instances from the console.

Note the above example has "YOUR PUBLIC KEY HERE" in the 'sshAuthorizedKeys' section. Change that to whatever your id_rsa.pub PKCS8 formatted public key is. (in the format `ssh-rsa AAAAB3NzaC1yc2EAAAA....==` )

Additional details for the node:

Recommended t2.large instance type
120GB root disk space
Make sure to deploy it to the cluster VPC's subnet (pick one of three subnets you created)


This node will strictly be where you compose the docker-registry and nginx node to serve as your Edge Registry and Ignition Webhost in order to deploy and strap your cluster nodes in the future. Once you deploy this node, SSH into it with the following command:

```
ssh -i /path/to/your/key core@<node ip>
```

'core' is the name of the username baked into the RHCOS image.

Once you've validated that the node can be connected to and managed over SSH, proceed to create a DNS record so outside clients can resolve this node.

### Create DNS Record
Create a DNS 'A' record in your private hosted zone with the corresponding certificate subject name to represent this registry.

If you named your registry `registry.mycluster.mydomain.tld` then create an A record named 'registry' that points to this node's IP address. The cert you upload in the next step must match that record name in order for TLS to work when you login and authenticate to the docker registry.

## Upload Certs and Auth components to Node
First, create a folder called "registry" as well as three folders inside of it to represent the volume mounts we will need to configure docker-registry accordingly:

```
mkdir -p registry/auth
mkdir -p registry/certs
mkdir -p registry/data
```

You should be left with a folder structure like so:
```
.
└── registry
    ├── auth
    ├── certs
    └── data
```

Now you will need to upload the htpasswd file and domain.crt+domain.key to the respective directories.

```
scp -i /path/to/key domain* core@node-ip:/path/to/certs
scp -i /path/to/key htpasswd core@node-ip:/path/to/auth/dir
```

## Upload and Import utility images to Node
SCP the aforementioned utility container images to the node
```
scp -i /path/to/key registry.tar core@<node ip>:/var/home/core/
scp -i /path/to/key nginx.tar core@<node ip>:/var/home/core/
```

ingest with podman:
```
podman load -i nginx.tar
podman load -i registry.tar
```

## Run docker-registry container which will hold the content
keep in mind you'll need to run this as root podman and --privileged in order to bind the lower ports accordingly. also make sure to update/edit the paths of the volume mounts so your htpasswd and certs/keys get mounted to the docker registry container correctly.
```
sudo podman run --privileged --name mirror-registry --restart=always -p 443:443 -v /var/home/core/registry/data:/var/lib/registry:z -v /var/home/core/registry/auth:/auth:z -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_HTTP_ADDR=0.0.0.0:443 -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v /var/home/core/registry/certs:/certs:z -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key -e REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true -d docker.io/library/registry:latest
```

Take note of how we bound the correct volume mounts to each respective function for docker-registry to work. `/var/home/core/registry/...` provides that structure for managing the certs your generated earlier as well as the htpasswd file you put together as well.

To validate your docker registry is functional, from your workstation on VPN, use `podman login <address of registry node using DNS FQDN>` to authenticate and login to the registry using the credentials you created with htpasswd and stored earlier.

## Login to registry and save pull secret
Log into your registry and save off the cached credentials to be able to craft your custom pull secret later:
```
podman login registry.mycluster.mydomain.tld
```

use the htpasswd creds you created to finish the login.

**If authentication or connection fails:**

Try some of the following:
1. Ensure the docker container you spun up is correctly binding port 443 on the host (i.e with `-e REGISTRY_HTTP_ADDR=0.0.0.0:443` in the podman command and `-p 443:443`)
2. Check the security group of the EC2 instance. allow port 443 and port 80 (HTTPS and HTTP)
3. Ensure pod was run as sudo and --privileged so it can bind ports lower than 1024. (in this case, 443)
4. Check if the cert matches the DNS record and hostname you're trying to login to

Once you've gotten a login succeeded message, you need to pull your cached credentials in order to concatenate them with your official Red Hat Cluster Manager Pull Secret:
```
cat /run/user/1000/containers/auth.json > auth.json
```

## Upload content to the edge registry
Using your new cached credentials from logging into your new registry instance with podman, you can now upload your offline image content to the docker registry:
```
oc image mirror -a auth.json --from-dir=mirror 'file://openshift/release:4.5.4*' registry.mycluster.mydomain.tld/openshift
```

Take note of the registry and repo you specify for use later when you perform the installation of Openshift. (in this case, `registry.mycluster.mydomain.tld/openshift` is our example)

**If image upload fails**:

1. Check if your auth.json is correct. You can pull it from /run/user/1000/containers/auth.json if you're using latest podman on your workstation.
2. Check if you still have network connectivity to the registry node from your workstation

## Create your special pull secret
Carefully concatenate the pull secret from Red Hat Cluster Manager with the contents from your auth.json cached credentials to your local registry instance you just created. This "super" pull secret will be used throughout the openshift install process.
Validate your pull secret is correctly stitched together with a json linter.

** THIS IS A CRITICAL STEP! DO NOT FORGET IT!**

Save a copy of your super pull secret that can access Quay as well as your custom offline edge registry somewhere safe.

# Ignition Webhost Composure

## Generate, Modify, and Bake out your .ign files


### Step 1: Install-Config
Start by generating your first artifact, the `install-config.yaml` template. This will be your high level "config" template that will be ingested to make a bunch of other artifacts to fine tune your cluster configs.
```
openshift-install create install-config --dir=<install dir>
# make sure you backup your install config after you're done mucking about with it. You don't want to have to fill this out multiple times.
cp install-dir/install-config.yaml ./backups/install-config.yaml
```

Modify this new install-config template like so:
```
additionalTrustBundle: |-
  -----BEGIN CERTIFICATE-----
  MIIGMTCCBBmgAwIBAgIUEhpa6/gdV83cP7x4p5eJ+vCU1CYwDQYJKoZIhvcNAQEL
  BQAwgacxCzAJBgNVBAYTAlVTMRMwEQYDVQQIDApXYXNoaW5ndG9uMRUwEwYDVQQH
  DAxEZWZhdWx0IENpdHkxEDAOBgNVBAoMB1JlZCBIYXQxDDAKBgNVBAsMA1NQTzEt
  MCsGA1UEAwwkcmVnaXN0cnkuZ292LWNsdXN0ZXIuZ292LnJodC1zcG8uY29tMR0w
  GwYJKoZIhvcNAQkBFg5zcG9AcmVkaGF0LmNvbTAeFw0yMDA3MjkxNzE1MDhaFw0y
  MTA3MjkxNzE1MDhaMIGnMQswCQYDVQQGEwJVUzETMBEGA1UECAwKV2FzaGluZ3Rv
  bjEVMBMGA1UEBwwMRGVmYXVsdCBDaXR5MRAwDgYDVQQKDAdSZWQgSGF0MQwwCgYD
  VQQLDANTUE8xLTArBgNVBAMMJHJlZ2lzdHJ5Lmdvdi1jbHVzdGVyLmdvdi5yaHQt
  c3BvLmNvbTEdMBsGCSqGSIb3DQEJARYOc3BvQHJlZGhhdC5jb20wggIiMA0GCSqG
  SIb3DQEBAQUAA4ICDwAwggIKAoICAQDFvwG4U5H+RWRAddcPPdjknDUCPoqBLJWu
  qDAH5cVRTlE4nUCUKGT8tAkqkeRZAnlFnjtT8VylTmFAyEf0spoYi8qHEN+wYn2R
  hSLz4+aSltxWfNoXoAfgEIfj/DdVUtfFGgSZcwwiZIbDO6ElCPLBZO5M0d+swBoW
  N8SEn93J79wNrmEKOry3PC3Oga2bWkKLJQ+CIzmsPgj5LhsfEx5rKF0imvMM39qR
  IZ9a1/9gMeQRGuqpSWG19PuDPJQPa7nvmS5Ppig7ooy89Q+n2n14rz3rhYpwWvLU
  FGmE5B3hQufl2i/swkNXIks0jxxPTTpFDrNaMrCNcBcoSchoZucrS6EVb2l7pPk1
  rUNnbSdd2S8tCMLdDOpU2EPxCPJkU6NwZ4V/8YExkzbkF3ZtGG33i8ZCxnSbtbuP
  Iw2jsOie+L+yr5IWXTO == NOT A REAL CERT ==    kIIXf3Fst8bLcLyEdWe
  gv9r6SMYtuprL5laSwU2xRXOH/+QOtyObXQVPDkAZoWGRQ2YBOoNpLUdmqv0zn8U
  yULPTti4zx ==                                     == qnTbF13HwUq
  7v7S9Somow == PASTE YOUR domain.crt CONTENTS HERE == kwDwYDVR0TA
  /zANBgkqhk ==                                     == rEJMWRaOf5j
  An77bb2s57za+9Af2HViVn9WfTF8DYLl2nAG/4P8yV93Itulp+qNzL6YolFwbCUB
  KuQZ1ZM9ExrEMfMzdwjsm9CcjVOs9vhC/woqK/kjxSdkIWkk5ovR58YGB3UQg9eK
  rdjRV2Qc13XJ2IY10qmZbEwB8j8qZHkSewUi9QxSb+gjhEmv8aP3y3/PcREz0nxV
  ogcai3z4aK2oepfM/uzP4aIXFtU9eO7fCL5bmOWgPLh33BLf2SR9NzrGc80N/aAI
  Bj7HGxD5z5OGEBINmRU8yK1yOh/9Dj2irhz5VvFwg9Tsh41xYZc1sJjZ4ctBqUWN
  F+cCNjIQXhyK0efSBPyr/ITmF7otPDhEL7mpluqPibx+BnF6DJ4LdQFAukZFzdqa
  AH+5bzegTtyM9rLPXZpjc5cdiFB1YJwoTENc0NsQx0ayMNABWM8vhVAi1ZnRppHM
  nzLmHRSXErpueGyggJr/hBGtLJqv7ozt8ZaMYonzJupbM/ZdvY2a0H3HcGMD4TkG
  Rga3q6tfzdzB4D2EvUSnGq6waL3UnGDRzyNmLh+wv/Ls1ABvrRMiGdrY0NQVFpAg
  rBPcpgQ+Hwyyunte9pdneYA4EHPIMyR/95VSDIedhQEH8IAeO3mY0bmtaGJxrnHA
  A37CSQA=
  -----END CERTIFICATE-----
apiVersion: v1
baseDomain: mydomain.tld
compute:
- hyperthreading: Enabled
  name: worker
  platform:
    aws:
      rootVolume:
        iops: 0
        size: 120
        type: gp2
      type: m5.2xlarge
      zones:
      - us-west-2a
      - us-west-2b
      - us-west-2c
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      rootVolume:
        iops: 0
        size: 120
        type: gp2
      type: m5.2xlarge
      zones:
      - us-west-2a
      - us-west-2b
      - us-west-2c
  replicas: 3
imageContentSources:
- mirrors:
  - registry.mycluster.mydomain.tld/openshift
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.mycluster.mydomain.tld/openshift
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
metadata:
  creationTimestamp: null
  name: mycluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    amiID: [YOUR RHCOS AMI ID]
    region: us-west-2
publish: Internal
pullSecret: 'paste your special pull secret in single quotes here'
sshKey: 'paste your SSH key in format "ssh-rsa abcdefhdksa3412==" here (PKCS8)'

```
Make sure to pay attention to some key settings in the above template:
1. `additionalTrustBundle` needs to be your `domain.crt` file so the cluster nodes can validate and trust the TLS connection to your docker registry.
    - be careful with the `|-` character after this, as its YAML syntax to preserve the entire cert bundle on multiple lines indented with the parameter.
2. `baseDomain` should match your `mydomain.tld` DNS zone BASE DOMAIN ONLY.
3. '`platform`' zones for both the workers and masters should be the zones you will use as a dummy for running the openshift installer, in this case we're using us-west-2 even though we plan to install to us-gov-west-1
        - Don't skimp on this. There is weird behavior if you don't specify platform details for the workers, such as your pull secret won't get pulled into the manifests.
4. `imageContentSources` needs to be formatted to point to your registry node, as well as the repo you used to upload your images from previous steps.
5. `metadata:name` should contain name similar to `mycluster` since this will determine the `*.apps.mycluster.mydomain.tld` apps k8s load balancer configuration
6. `platform:aws:amiID` should be the RHCOS AMI ID you produced in earlier steps.
7. `publish` needs to be set to `Internal` in order for the cluster not to try to look for and produce public hosted zone records
8. `pullSecret` needs to be your special concatenated pull secret
9. `sshKey` needs to be your public RSA fingerprint of your desired SSH key in PKCS 8 format, or "`ssh-rsa ABCDabcd1234==`" format

**ONCE YOU ARE DONE FORMATTING YOUR INSTALL-CONFIG.YAML FILE, BACK IT UP!!!**
It really sucks having to fill this out more than once, and the installer eats the file when it generates manifests. So if you need to redo this step, which you most likely will, it's easier to just start from a backup template.


### Step 2: Manifests
Now that you have your install-config template, you need to consume it and generate manifests based on it using the installer.
```
openshift-install create manifests --dir=<install dir>
# then you modify the 20+ manifests and their DNS, regions, etc since the installer doesn't support this stuff.
# again, back up your manifests if you realize you made a mistake and want to regenerate .ign files from them, but remember that they expire in 24 hours from the certs that the installer binary generates.
cp -r <install dir> ./backups/
```
Performing this step will destroy the install-config.yaml file and produce roughly 20-25 manifests that you will now need to comb through and modify to fit your private region in govcloud.

While combing through the manifests make these general modifications:

1. change any instance of `mycluster-abcd123-region-az` to just `mycluster-region-az` since the installer automatically generates some random characters to differentiate clusters and their naming standards
2. change all instances of the public/commercial AWS region to the private region you're targeting (e.g. change `us-west-2` occurrences in the manifests to `us-gov-west-1` or similar)
3. DELETE all machineset manifests for masters, as you will be providing your own by hand. I think you can leave the workers machineset manifests, since the masters will run operators that can interact with the IaaS platform and spin them up automatically as long as the IAM keys added are valid

WIP: Need to add steps that talk about creating 3 whole operator IAM accounts and corresponding keys, since this doesn't seem to create usable keys out of the box. (THIS IS OUR CURRENT FAILURE POINT)


### Step 3: Ignition Files
Once you're done performing surgery on the manifests, you can then compile them into .ign files using the installer. Again, you may want to backup the manifests first if you don't want to lose them for good.
```
openshift-install create ignition-configs --dir=<install dir>
# generate the final .ign payloads. you will use these files in your ignition webhost.
```
This final step will transpile the manifests into .ign files which you will proceed to upload to your Ignition Webhost in order to deploy your final nodes to ingest remote configs from.

## SCP Ignition Files to your webhost
It's recommended to just use the same registry node you used to build your docker-registry, or you can use S3 as an alternative.
```
scp -i /path/to/key *.ign core@node-ip:/path/to/html/directory
```

## Run your NGINX Container

CHMOD your html directory where your files are at.
```
sudo chmod -R 775 /path/to/html/directory
```

keep in mind we're running this as root/privileged again in order to bind the lower ports. Make sure to update/modify the path for your html directory accordingly.
```
sudo podman run -d --rm --privileged -p 80:80 --name=nginx -v /var/home/core/nginx/html:/usr/share/nginx/html docker.io/library/nginx:latest
```

validate your ignition files are good to go.
```
curl http://node-ip/bootstrap.ign
curl http://node-ip/master.ign
curl http://node-ip/worker.ign
```

if you can't get a response, then you messed up either the selinux, facls of the files/html directory or you misformatted your podman command or didn't run it as sudo.


## Bootstrap Node Deployment

Deploy your bootstrap node by deploying off the RHCOS AMI you uploaded to your tenancy, then ensure the following details are accounted for:
- Attach IAM Profile "mycluster-master-profile"
create your custom user data string for your bootstrap node. modify the source url accordingly. We tend to put both nginx and docker-registry on the same node, so you could in theory just use that FQDN.
```
{"ignition":{"config":{"append":[{"source":"http://<nginxhost>.mycluster.mydomain.tld/bootstrap.ign","verification":{}}]},"security":{},"timeouts":{},"version":"2.2.0"},"networkd":{},"passwd":{},"storage":{},"systemd":{}}
```

