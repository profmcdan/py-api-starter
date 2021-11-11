# Setup AWS KOPS

## Set necessary env variable and create group

```bash
AWS_ACCESS_KEY_ID=xxx
export AWS_SECRET_ACCESS_KEY=xxx
export AWS_DEFAULT_REGION=us-east-2
aws iam create-group --group-name kops
```

The response is as follows

```json
{
    "Group": {
        "Path": "/",
        "GroupName": "kops",
        "GroupId": "AGPAX43GMNUGJ3EL62GJE",
        "Arn": "arn:aws:iam::542991215884:group/kops",
        "CreateDate": "2021-10-23T09:43:17+00:00"
    }
}
```

Next, we’ll assign a few policies to the group thus providing the future users of the group with sufficient permissions to create the objects we’ll need. Since our cluster will consist of EC2 instances, the group will need to have the permissions to create and manage them. We’ll need a place to store the state of the cluster so we’ll need access to S3. Furthermore, we need to add VPCs to the mix so that our cluster is isolated from prying eyes. Finally, we’ll need to be able to create additional IAMs.

```bash
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
```

## Create a user

Now that we have a group with the sufficient permissions, we should create a user as well

```bash
aws iam create-user --user-name kops
```

The response is as follows:

```json
{
    "User": {
        "Path": "/",
        "UserName": "kops",
        "UserId": "AIDAX43GMNUGF4SXFLQSL",
        "Arn": "arn:aws:iam::542991215884:user/kops",
        "CreateDate": "2021-10-23T09:50:49+00:00"
    }
}
```

The user we created does not yet belong to the kops group, add the user to the group

```bash
aws iam add-user-to-group --user-name kops --group-name kops
```

Finally, we’ll need access keys for the newly created user. Without them, we would not be able to act on its behalf.

```bash
aws iam create-access-key --user-name kops > aws-kops-creds
cat aws-kops-creds
```

We need the SecretAccessKey and AccessKeyId entries. So, the next step is to parse the content of the aws-kops-creds file and store those two values as the environment variables AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY.

In the spirit of full automation, we’ll use jq to parse the contents of the aws-kops-creds file. Please download and install the distribution suited for your OS.

```bash
export AWS_ACCESS_KEY_ID=$(cat aws-kops-creds | jq -r '.AccessKey.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(cat aws-kops-creds | jq -r '.AccessKey.SecretAccessKey')
```

We used cat to output contents of the file and combined it with jq to filter the input so that only the field we need is retrieved.
From now on, all the AWS CLI commands will not be executed by the administrative user you used to register to AWS, but as kops

## Availability zones setup

```bash
aws ec2 describe-availability-zones --region $AWS_DEFAULT_REGION
```

The output

```json
{
    "AvailabilityZones": [
        {
            "State": "available",
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "RegionName": "us-east-2",
            "ZoneName": "us-east-2a",
            "ZoneId": "use2-az1",
            "GroupName": "us-east-2",
            "NetworkBorderGroup": "us-east-2",
            "ZoneType": "availability-zone"
        },
        {
            "State": "available",
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "RegionName": "us-east-2",
            "ZoneName": "us-east-2b",
            "ZoneId": "use2-az2",
            "GroupName": "us-east-2",
            "NetworkBorderGroup": "us-east-2",
            "ZoneType": "availability-zone"
        },
        {
            "State": "available",
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "RegionName": "us-east-2",
            "ZoneName": "us-east-2c",
            "ZoneId": "use2-az3",
            "GroupName": "us-east-2",
            "NetworkBorderGroup": "us-east-2",
            "ZoneType": "availability-zone"
        }
    ]
}
```

As we can see, the region has three availability zones. We’ll store them in an environment variable.

For windows user:
Please use ```tr '\r\n' ', '``` instead of ```tr '\n' ','``` in the command that follows.

```bash
export ZONES=$(aws ec2 describe-availability-zones --region $AWS_DEFAULT_REGION | jq -r '.AvailabilityZones[].ZoneName' | tr '\n' ',' | tr -d ' ')
ZONES=${ZONES%?}
echo $ZONES
```

Just as with the access keys, we used jq to limit the results only to the zone names, and we combined that with tr that replaced new lines with commas. The second command removes the trailing comma. The output of the last command that echoed the values of the environment variable

## Create SSH Key

We create a new key pair, filtered the output so that only the KeyMaterial is returned, and stored it in the devops23.pem file. For security reasons, we should change the permissions of the devops23.pem file so that only the current user can read it. Finally, we’ll need only the public segment of the newly generated SSH key, so we’ll use ssh-keygen to extract it

```bash
aws ec2 create-key-pair --key-name devops23 | jq -r '.KeyMaterial' > devops23.pem
chmod 400 devops23.pem
ssh-keygen -y -f devops23.pem > devops23.pub
```

All those steps might look a bit daunting if this is your first contact with AWS. Nevertheless, they are pretty standard. No matter what you do in AWS, you’d need to perform, more or less, the same actions. Not all of them are mandatory, but they are good practices. Having a dedicated (non-admin) user and a group with only required policies is always a good idea. Access keys are necessary for any aws command. Without SSH keys, no one can interactively log in to a server.

The good news is that we’re finished with the prerequisites. In the next lesson, we can turn our attention towards creating a Kubernetes cluster.

## Create s3 and install kops

We’ll start by deciding the name of our soon to be created cluster. We’ll choose to call it devops23.k8s.local. The latter part of the name (.k8s.local) is mandatory if we do not have a DNS at hand. It’s a naming convention kops uses to decide whether to create a gossip-based cluster or to rely on a publicly available domain.

If this would be a “real” production cluster, you would probably have a DNS for it, or if you have your own DNS and you are planning to use it to set-up cluster, there will be minor additional steps to the things we have done so far, please refer to Kops for exact steps.

However, since we cannot be sure whether you do have one for the exercises in this course, we’ll play it safe, and proceed with the gossip mode.

We’ll store the name in an environment variable so that it is easily accessible.

```bash
export KOPS_DEV_NAME=devops23.k8s.local
```

When we create the cluster, kops will store its state in a location we’re about to configure. kops uses the state it generates when creating the cluster for all subsequent operations. If we want to change any aspect of a cluster, we’ll have to change the desired state first, and then apply those changes to the cluster.

```bash
export BUCKET_NAME=devops23-$(date +%s)
aws s3api create-bucket --bucket $BUCKET_NAME --create-bucket-configuration LocationConstraint=$AWS_DEFAULT_REGION
```

Output is:

```json
{
    "Location": "http://devops23-1634985168.s3.amazonaws.com/"
}
```

For simplicity, we’ll define the environment variable KOPS_STATE_STORE. Kops will use it to know where we store the state. Otherwise, we’d need to use --store argument with every kops command.

```bash
export KOPS_STATE_STORE=s3://$BUCKET_NAME
```

On Mac:

```bash
brew update && brew install kops
kops version
```

### Making Choice with Master Nodes

The first question we might ask ourselves is whether we want to have high-availability. It would be strange if anyone would answer no. Who doesn’t want to have a cluster that is (almost) always available? Instead, we’ll ask ourselves what are the things that might bring our cluster down.

When a node is destroyed, Kubernetes will reschedule all the applications that were running inside it into the healthy nodes. All we have to do is to make sure that, later on, a new server is created and joined the cluster, so that its capacity is back to the desired values. We’ll discuss later how are new nodes created as a reaction to failures of a server. For now, we’ll assume that will happen somehow.

Still, there is a catch. Given that new nodes need to join the cluster, if the failed server was the only master, there is no cluster to join. All is lost. The most important part is where the master servers are. They host the critical components without which Kubernetes cannot operate.

So, we need more than one master node. How about two? If one fails, we still have the other one. Still, that would not work.

Every piece of information that enters one of the master nodes is propagated to the others, and only after the majority agrees, that information is committed. If we lose majority (50%+1), masters cannot establish a quorum and cease to operate. If one out of two masters is down, we can get only half of the votes, and we would lose the ability to establish the quorum. Therefore, we need three masters or more. Odd numbers greater than one are “magic” numbers. Given that we won’t create a big cluster, three should do.

With three masters, we are safe from a failure of any single one of them. Given that failed servers will be replaced with new ones, as long as only one master fails at the time, we should be fault tolerant and have high availability.

`Always set an odd number greater than one for master nodes.`

### Making Choice with Data Centers

The whole idea of having multiple masters does not mean much if an entire data center goes down.

Attempts to prevent a data center from failing are commendable. Still, no matter how well a data center is designed, there is always a scenario that might cause its disruption. So, we need more than one data center. Following the logic behind master nodes, we need at least three. But, as with almost anything else, we cannot have any three (or more) data centers. If they are too far apart, the latency between them might be too high. Since every piece of information is propagated to all the masters in a cluster, slow communication between data centers would severely impact the cluster as a whole.

All in all, we need three data centers that are close enough to provide low latency, and yet physically separated, so that failure of one does not impact the others. Since we are about to create the cluster in AWS, we’ll use availability zones (AZs) which are physically separated data centers with low latency.
`Always spread your cluster between at least three data centers which are close enough to warrant low latency.`
There’s more to high-availability than running multiple masters and spreading a cluster across multiple availability zones. We’ll get back to this subject later. For now, we’ll continue exploring the other decisions we have to make.

### Making Choice with Networking

Which networking shall we use? We can choose any of the following networkings:

* kubenet
* CNI
* classic
* external
The classic Kubernetes native networking is deprecated in favor of kubenet, so we can discard it right away.

The external networking is used in some custom implementations and for particular use cases, so we’ll discard that one as well.

That leaves us with kubenet and CNI.

Container Network Interface (CNI) allows us to plug in a third-party networking driver. Kops supports Calico, flannel, Canal (Flannel + Calico), kopeio-vxlan, kube-router, romana, weave, and amazon-vpc-routed-eni networks. Each of those networks comes with pros and cons and differs in its implementation and primary objectives. Choosing between them would require a detailed analysis of each. We’ll leave a comparison of all those for some other time and place. Instead, we’ll focus on kubenet.

Kubenet is kops’ default networking solution. It is Kubernetes native networking, and it is considered battle tested and very reliable. However, it comes with a limitation. On AWS, routes for each node are configured in AWS VPC routing tables. Since those tables cannot have more than fifty entries, kubenet can be used in clusters with up to fifty nodes. If you’re planning to have a cluster bigger than that, you’ll have to switch to one of the previously mentioned CNIs.

`Use kubenet networking if your cluster is smaller than fifty nodes.`

The good news is that using any of the networking solutions is easy. All we have to do is specify the --networking argument followed with the name of the network.
Given that we won’t have the time and space to evaluate all the CNIs, we’ll use kubenet as the networking solution for the cluster we’re about to create.

### Making Choice with the Nodes’ Size

Finally, we are left with only one more choice we need to make. What will be the size of our nodes? Since we won’t run many applications, t2.small should be more than enough and will keep AWS costs to a minimum. t2.micro is too small, so we elected the second smallest among those AWS offers.


## Create a cluster

```bash
kops create cluster --name $KOPS_DEV_NAME --master-count 3 --node-count 1 --node-size t2.small \
    --master-size t2.small --zones $ZONES --master-zones $ZONES \
    --ssh-public-key devops23.pub --networking kubenet --yes

kops get cluster
kubectl cluster-info
kops validate cluster

```

Suggestions:

* validate cluster: kops validate cluster --wait 10m
* list nodes: kubectl get nodes --show-labels
* ssh to the master: ssh -i ~/.ssh/id_rsa ubuntu@api.devops23.k8s.local
* the ubuntu user is specific to Ubuntu. If not using Ubuntu please use the appropriate user based on your OS.
* read about installing addons at: https://kops.sigs.k8s.io/operations/addons.

## Add a LoadBalancer to access out Node

```bash
kubectl create -f https://raw.githubusercontent.com/kubernetes/kops/master/addons/ingress-nginx/v1.6.0.yaml
```

if this fails ...
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.4/deploy/static/provider/aws/deploy.yaml
```

