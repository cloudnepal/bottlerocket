# Using a Bottlerocket AMI with Amazon EKS

The first release of Bottlerocket focuses on Kubernetes, in particular serving as the host OS for Kubernetes pods.

One easy way to get started is to use Amazon EKS, a service that manages a Kubernetes control plane for you.
This document will focus on EKS to make it easy to follow a single path.
There's nothing that limits Bottlerocket to EKS or AWS, though.

Most of this is one-time setup, and yes, we plan to automate more of it!
Once you have a cluster, you can skip to the last step, [Launch!](#launch)

## Dependencies

EKS has a command-line tool called `eksctl` that makes cluster setup easy.
Versions of eksctl starting with 0.15.0-rc.2 support Bottlerocket natively.
We recommend that you download the [latest version of eksctl](https://github.com/weaveworks/eksctl/releases) to get this support.

You'll also need to [install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to augment `eksctl` during setup, and to run pods afterward.

Finally, you'll need [aws-cli](https://aws.amazon.com/cli/) set up to interact with AWS.
(You'll need a [recent v1 release](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html#install-tool-bundled) with EKS support.)

## Automated setup

If you have a recent `eksctl`, as mentioned above, most of Bottlerocket setup for EKS is automated.

### Cluster setup

#### Cluster setup configuration file

eksctl can use a configuration file to simplify setup.
We have sample configuration files in the repo:
* [`sample-eksctl.yaml`](sample-eksctl.yaml) - recommended for most setups.
* [`sample-eksctl-ssh.yaml`](sample-eksctl-ssh.yaml) - for test clusters where you know you'll want SSH access.  Make sure to change the `publicKeyName` setting to the name of the SSH key pair you have registered with EC2.

Pick the file most appropriate for you and make a copy, for example `my-eksctl.yaml`.
In this file you can change your desired numbered of nodes and even set Bottlerocket settings in advance if you like.  The 'settings' section under 'bottlerocket' can include any [Bottlerocket settings](https://github.com/bottlerocket-os/bottlerocket#description-of-settings).

Note that the configuration file includes the AWS region, so change it from `us-west-2` if you operate in another region.

To learn more about eksctl configuration files, you can look at the [full schema](https://eksctl.io/usage/schema/) or [official examples](https://github.com/weaveworks/eksctl/tree/master/examples).

#### Cluster creation

You can set up a new cluster like this, pointing to the file you created in the last step:

```shell
eksctl create cluster --config-file ./my-eksctl.yaml
```

This will take a few minutes to create the EKS cluster and spin up your Bottlerocket worker nodes.

#### Optional cluster configuration

##### CSI plugin

If you want to create a [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) on a Bottlerocket host, you will need to use the [EBS CSI Plugin](https://github.com/kubernetes-sigs/aws-ebs-csi-driver).
This is because the default EBS driver relies on file system tools that are not included with Bottlerocket.
A walk-through of creating a storage class using the driver is available [here](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

##### conntrack configuration

By default `kube-proxy` will set the `nf_conntrack_max` kernel parameter to a default value that may differ from what Bottlerocket originally sets at boot.
If you prefer to keep Bottlerocket's [default setting](packages/release/release-sysctl.conf), edit the kube-proxy-config ConfigMap with:

```shell
kubectl edit -n kube-system cm/kube-proxy-config
```

Change the `maxPerCore` and `min` fields for `conntrack` like so (a setting of 0 implies no change):

```yaml
conntrack:
  maxPerCore: 0
  min: 0
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
```

### Done!

Bottlerocket instances are launched in an autoscaling group, up to the number specified in your eksctl configuration file.
(You can change this number after creation by [configuring the ASG](https://console.aws.amazon.com/ec2/autoscaling/home#AutoScalingGroups:view=details), the same way you might change other ASGs.)

The Bottlerocket instances will automatically register into the EKS cluster created by eksctl.
You can now use normal Kubernetes tools like `kubectl` to manage your cluster and the Bottlerocket nodes.

For example, to run a simple busybox pod:
`kubectl run -i -t busybox --image=busybox --restart=Never`

## Manual setup

If you'd like even more control over your setup, something that eksctl can't (yet) provide, or you just want to see what's involved, you can follow these steps.

### Finding an AMI

The official AMI IDs are stored in [public SSM parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-public-parameters.html).
The parameter names look like this: `/aws/service/bottlerocket/aws-k8s-1.24/x86_64/latest/image_id`

Just change the variant (`aws-k8s-1.24`) and architecture (`x86_64`) to the ones you want to use.
Supported variants and architectures are described in the [README](README.md#variants).
For the purposes of SSM parameters, the valid architecture names are `x86_64` and `arm64` (also known as `aarch64`).
Also, if you know a specific Bottlerocket version you'd like to use, for example `1.11.0`, you can replace `latest` with that version.

Bottlerocket EKS variants with NVIDIA support append `-nvidia` to the variant name.
For instance, the variant for Kubernetes version 1.28 with NVIDIA support is `aws-k8s-1.28-nvidia`.

Once you have the parameter name you want to use, the easiest way to use it is to pass it directly to EC2.
(You can also use this method for CloudFormation and other services that launch EC2 instances for you.)
Just prefix the parameter name with `resolve:ssm:` and EC2 will fetch the current value for you.

For example, to use the parameter above, you would pass this as the AMI ID in your launch request: `resolve:ssm:/aws/service/bottlerocket/aws-k8s-1.24/x86_64/latest/image_id`

#### Manually querying SSM

If you prefer to fetch the AMI ID yourself, you can use [aws-cli](https://aws.amazon.com/cli/) on the command line.
To fetch the example parameter above, for the us-west-2 region, you could run this:

```shell
aws ssm get-parameter --region us-west-2 --name "/aws/service/bottlerocket/aws-k8s-1.24/x86_64/latest/image_id" --query Parameter.Value --output text
```

If you have `jq` installed and would like a bit more information, try this:

```shell
aws ssm get-parameters --region us-west-2 \
   --names "/aws/service/bottlerocket/aws-k8s-1.24/x86_64/latest/image_id" \
           "/aws/service/bottlerocket/aws-k8s-1.24/x86_64/latest/image_version" \
   --output json | jq -r '.Parameters | .[] | "\(.Name): \(.Value) (updated \(.LastModifiedDate | gmtime | strftime("%c")) UTC)"'
```

### Cluster setup

*Note:* most commands will have a region argument; make sure to change it if you don't want to set up in us-west-2.
Also be aware that when operating in GovCloud the IAM ARNs will need to be updated to the following: `arn:aws-us-gov`.
For example, `arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy` must be updated to
`arn:aws-us-gov:iam::aws:policy/AmazonEKSWorkerNodePolicy`.

You can set up a new cluster like this:

```shell
eksctl create cluster --region us-west-2 --name bottlerocket
```

This will automatically add a "context" so `kubectl` knows how to interact with your cluster, and it'll set that context as your default.
You can see your contexts (clusters) using `kubectl config get-contexts` and change your current one with `kubectl config use-context 'NEW-CONTEXT-HERE'`.

### Cluster info

This section helps you determine some of the cluster information needed later by the instance launch command.

#### Kubernetes cluster info

Bottlerocket uses a TOML-formatted configuration file as user data.
This can include the configuration of the Kubernetes cluster we just created.

Run this to generate the configuration file with the relevant cluster config, including the API endpoint and base64-encoded certificate authority.

```shell
eksctl get cluster --region us-west-2 --name bottlerocket -o json \
   | jq --raw-output '.[] | "[settings.kubernetes]\napi-server = \"" + .Endpoint + "\"\ncluster-certificate =\"" + .CertificateAuthority.Data + "\"\ncluster-name = \"bottlerocket\""' > user-data.toml
```

This will save the TOML-formatted configuration data into a file named `user-data.toml`.
This will be used at the end, in the instance launch command.

#### Subnet info

Next, run this to get information about the subnets that eksctl created.
It will give you a list of the subnets and tell you whether each is public or private.
(If you use an EC2 region other than "us-west-2", make sure to change that.)

```shell
aws ec2 describe-subnets \
   --subnet-ids $(eksctl get cluster --region us-west-2 --name bottlerocket -o json | jq --raw-output '.[].ResourcesVpcConfig.SubnetIds[]') \
   --region us-west-2 \
   --query "Subnets[].[SubnetId, Tags[?Key=='aws:cloudformation:logical-id'].Value]" \
   | xargs -L2
```

You'll want to pick one and save it for the launch command later.

You can choose whether you want public or private.
* Choose private for production deployments to get maximum isolation of worker nodes.
* Choose public to more easily debug your instance.  These subnets have an Internet Gateway, so if you add a public IP address to your instance, you can talk to it.  (You can manually add an Internet Gateway to a private subnet later, so this is a reversible decision.)

Note that if you choose to use the public subnet, you'll need your instance to have a publicly accessible IP address.
That either means adding `--associate-public-ip-address` to the launch command below, or attaching an Elastic IP address after launch.
There will be a reminder about this when we talk about the launch command.

Finally, note that if you want to launch in a specific availability zone, make sure you pick a subnet that matches; the AZ is listed right next to the public/private status.

### IAM role

The instance we launch needs to be associated with an IAM role that allows for communication with EKS and ECR.

`eksctl` by default already creates such a role (and an instance profile that allows use of the role) as part of the cluster nodegroup.

The ARN of the IAM role can be retrieved with:

```shell
eksctl get iamidentitymapping --region us-west-2 --cluster bottlerocket
```

The output should look like this:

```
ARN                                                               USERNAME                                GROUPS
arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/INSTANCE_ROLE_NAME          system:node:{{EC2PrivateDNSName}}       system:bootstrappers,system:nodes
```

Note down the INSTANCE_ROLE_NAME for the instructions below.

##### Enabling SSM

If you add SSM permissions, you can use Bottlerocket's default SSM agent to get a shell session on the instance.

To attach the role policy for SSM permissions, run the following:

```shell
aws iam attach-role-policy \
   --role-name INSTANCE_ROLE_NAME \
   --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

If you receive the following error, you need to truncate INSTANCE_ROLE_NAME to 64 characters.
(We are working on improving this.)

```
1 validation error detected: Value 'INSTANCE_ROLE_NAME' at 'role Name' failed to satisfy constraint:
Member must have length less than or equal to 64
```

Next, to retrieve the instance profile name used to launch instances, run this:

```shell
aws iam list-instance-profiles-for-role --role-name INSTANCE_ROLE_NAME --query "InstanceProfiles[*].InstanceProfileName" --output text
```

There should only be one that looks like:

```
eksctl-bottlerocket-nodegroup-ng-IDENTIFIER-NodeInstanceProfile-IDENTIFIER
```

Note this down as the `INSTANCE_PROFILE_NAME` for the final launch command.

### kube-proxy settings

By default `kube-proxy` will set the `nf_conntrack_max` kernel parameter to a default value that may differ from what Bottlerocket originally sets at boot.
If you prefer to keep Bottlerocket's [default setting](packages/release/release-sysctl.conf), edit the kube-proxy configuration details with:

```shell
kubectl edit -n kube-system daemonset kube-proxy
```

Add `--conntrack-max-per-core` and `--conntrack-min` to the kube-proxy arguments like so (a setting of 0 implies no change):

```yaml
      containers:
      - command:
        - kube-proxy
        - --v=2
        - --config=/var/lib/kube-proxy-config/config
        - --conntrack-max-per-core=0
        - --conntrack-min=0

```

### Final launch details

For the instance to be able to communicate with the EKS cluster control plane and other worker nodes, we need to make sure the instance is launched with the right security groups.

Run the following command:

```shell
aws ec2 describe-security-groups --region us-west-2 \
  --filters 'Name=tag:Name,Values=*bottlerocket*' \
  --query "SecurityGroups[*].{Name:GroupName,ID:GroupId}"
```

This will output several security group names and IDs.
You want to save the IDs for the `...ClusterSharedNodeSecurityGroup...` and `...nodegroup...` entries.

Example:

```json
[
    {
        "Name": "eksctl-bottlerocket-cluster-ClusterSharedNodeSecurityGroup-IDENTIFIER",
        "ID": "SECURITY_GROUP_ID_1"
    },
    {
        "Name": "eksctl-bottlerocket-cluster-ControlPlaneSecurityGroup-IDENTIFIER",
        "ID": *ignore*
    },
    {
        "Name": "eksctl-bottlerocket-nodegroup-ng-IDENTIFIER-SG-IDENTIFIER",
        "ID": "SECURITY_GROUP_ID_2"
    }
]
```

If you chose a public subnet, and you plan to SSH to the instance (using the admin container), you'll also need to allow SSH traffic to your security group.
You can do that with a command like this - just make sure to insert a security group from the last command, and your source network CIDR.

```shell
aws ec2 authorize-security-group-ingress --region us-west-2 \
   --group-id SECURITY_GROUP_ID_1 --cidr YOUR_NETWORK_CIDR \
   --protocol tcp --port 22
```

If you chose a private subnet and you want to SSH in, you can do so from another instance in the same subnet and security group.

### Launch!

Now we can launch a Bottlerocket instance in our cluster!

There are a few values to make sure you change in this command:
* YOUR_KEY_NAME: your SSH key pair name, as registered with EC2
* SUBNET_ID: the subnet you selected earlier
  * If you chose a public subnet, either add `--associate-public-ip-address` to the command, or attach an Elastic IP afterward.
* SECURITY_GROUP_ID_1, SECURITY_GROUP_ID_2: the two security groups you found earlier
* BOTTLEROCKET_AMI_ID: the ID of the AMI you registered, or an Amazon-provided AMI ID
* user-data.toml: the path to the user data file you created earlier
* INSTANCE_PROFILE_NAME: the instance profile created by `eksctl` for the cluster nodegroups.

```shell
aws ec2 run-instances --key-name YOUR_KEY_NAME \
   --subnet-id SUBNET_ID \
   --security-group-ids SECURITY_GROUP_ID_1 SECURITY_GROUP_ID_2 \
   --image-id BOTTLEROCKET_AMI_ID \
   --instance-type c3.large \
   --region us-west-2 \
   --tag-specifications 'ResourceType=instance,Tags=[{Key=kubernetes.io/cluster/bottlerocket,Value=owned}]' \
   --user-data file://user-data.toml \
   --iam-instance-profile Name=INSTANCE_PROFILE_NAME
```

And remember, if you used a public subnet, add `--associate-public-ip-address` or attach an Elastic IP after launch.

Once it launches, you should be able to run pods on your Bottlerocket instance using normal Kubernetes workflows.

For example, to run busybox:
`kubectl run -i -t busybox --image=busybox --restart=Never`

### aws-k8s-*-nvidia variants

The `aws-k8s-*-nvidia` variants include the required packages and configurations to leverage NVIDIA GPUs.
They come with the [NVIDIA Tesla driver](https://docs.nvidia.com/datacenter/tesla/drivers/index.html) along with the libraries required by the [NVIDIA container runtime](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit).
They also include the [NVIDIA k8s device plugin](https://github.com/NVIDIA/k8s-device-plugin).
If you already have a daemonset for the device plugin in your cluster, you may need to use taints and tolerations to keep it from running on Bottlerocket nodes.

Additional NVIDIA tools such as [DCGM exporter](https://github.com/NVIDIA/dcgm-exporter) and [GPU Feature Discovery](https://github.com/NVIDIA/gpu-feature-discovery) will work as expected.
You can install them in your cluster by following the `helm install` instructions provided for each project.

The [GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#install-nvidia-gpu-operator) can also be used to install these tools.
However, it is cumbersome to select the right subset of features to avoid conflicts with the software included in the variant.
Therefore we recommend installing the tools individually if they are required.

In hosts with multiple GPUs (ex. EC2 `g4dn` instances) you can assign a GPU per container by specifying the resource in the containers' spec as described in the [official kubernetes documentation](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  restartPolicy: OnFailure
  containers:
    - name: test
      image: amazonlinux:2
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
```
