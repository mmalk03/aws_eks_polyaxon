# AWS EKS Polyaxon
This file explains how to setup Amazon EKS (Elastic Kubernetes Service) on AWS.
Then, it describes how to deploy Polyaxon (platform to manage machine learning experiments at scale) to the Kubernetes cluster using Helm chart and establish connection from local machine.

## Conventions
Command prefixes:
* `$` - run on local machine
* `ec2-user$` - run on AWS machine

## Install AWS CLI
```bash
$ pacman -S aws-cli
$ aws configure
```
For detailed instructions look [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html).
Optionally, you may also install [aws-shell](https://github.com/awslabs/aws-shell), an interactive shell for the AWS CLI.

## Stack deployment

#### Create an Amazon EC2 Key Pair
To enable ssh connection with created VMs you need to use an existing Key Pair or create a new one.
To create new Key Pair with name `eks-key-pair` (has to be unique) run:
```bash
$ aws ec2 create-key-pair --key-name eks-key-pair --query 'KeyMaterial' --output text > ~/.aws/eks-private-key.pem
$ chmod 400 ~/.aws/eks-private-key.pem
```

#### Deploy the stack
```bash
$ aws cloudformation deploy \
    --template-file amazon-eks-master.template.yaml \
    --stack-name eks-stack \
    --parameter-overrides AvailabilityZones=eu-west-2a,eu-west-2b,eu-west-2c KeyPairName=eks-key-pair RemoteAccessCIDR=0.0.0.0/0
```
For detailed instructions look [here](https://s3.amazonaws.com/aws-quickstart/quickstart-amazon-eks/doc/amazon-eks-architecture.pdf).

## Connect to Kubernetes
Created stack contains bastion of Linux VMs, which have `kubectl` to interact with Kubernetes cluster.
They also have `helm` to manage Kubernetes packages, including **Polyaxon**.
Find the public IP of one of the VMs:
```bash
$ IP=$(
    aws ec2 describe-instances \
        --filter Name=tag:Name,Values=LinuxBastion \
        --query "Reservations[].Instances[].PublicIpAddress" \
        --output text
)
```
Connect to it using ssh:
```bash
$ ssh -i ~/.aws/eks-private-key.pem ec2-user@${IP}
```
Test kubectl and helm:
```bash
ec2-user$ kubectl version
ec2-user$ helm version
```

## Deploy Polyaxon

#### Install from Helm chart
```bash
ec2-user$ helm repo add polyaxon https://charts.polyaxon.com
ec2-user$ helm repo update
ec2-user$ helm install --name=polyaxon --namespace=polyaxon --wait polyaxon/polyaxon
```

#### Get instructions how to setup connection from local machine
```bash
ec2-user$ POLYAXON_IP=$(
    kubectl get svc \
        --namespace polyaxon polyaxon-polyaxon-api \
        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
)
ec2-user$ POLYAXON_ROOT_PASSWORD=$(
    kubectl get secret \
        --namespace polyaxon polyaxon-polyaxon-secret \
        -o jsonpath="{.data.POLYAXON_ADMIN_PASSWORD}" \
        | base64 --decode
)
ec2-user$ echo "Polyaxon dashboard is available at: ${POLYAXON_IP}"
ec2-user$ echo "To setup polyaxon from local machine run:
    polyaxon config set --host=${POLYAXON_IP} --http_port=80 --ws_port=1337
    polyaxon login
"
ec2-user$ echo "USER: root"
ec2-user$ echo "PASSWORD: ${POLYAXON_ROOT_PASSWORD}"
```

#### Install Polyaxon on local machine
```bash
$ pip install -U polyaxon-cli
```

#### Setup connection from local machine
```bash
$ polyaxon config set --host=<as-printed-above> --http_port=80 --ws_port=1337
$ polyaxon login --username root --password rootpassword
```

#### Verify connection
```bash
$ polyaxon cluster
$ polyaxon project create --name=quick-start --description='Polyaxon quick start.'
```

## Cleanup

#### Uninstall Helm chart
```bash
ec2-user$ helm delete polyaxon --purge
```

#### Delete stack
```bash
$ aws cloudformation delete-stack --stack-name eks-stack
```
