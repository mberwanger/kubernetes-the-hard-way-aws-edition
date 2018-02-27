# Prerequisites

## Amazon Web Services

This tutorial leverages the [Amazon Web Services](https://aws.amazon.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://aws.amazon.com/free/) for the AWS Free Tier.

[Estimated cost](https://calculator.s3.amazonaws.com/index.html#r=IAD&s=EC2&key=calc-ED0A6094-FA45-44DC-9CCB-535A6F4BF337) to run this tutorial: $0.17 per hour ($4.05 per day).

> The compute resources required for this tutorial exceed the Amazon Web Services free tier.

## AWS Command Line Interface

### Installing the AWS Command Line Interface

Follow the installing the AWS command line interface
 [documentation](https://docs.aws.amazon.com/cli/latest/userguide/installing.html) to install the `aws` command line utility.

 Requirements
 - Python 2 version 2.6.5+ or Python 3 version 3.3+
 - Windows, Linux, macOS, or Unix

   *Note - Older versions of Python may not work with all AWS services. If you see InsecurePlatformWarning or deprecation notices when you install or use the AWS CLI, update to a recent version.*

Verify the AWS CLI version is 1.11.0 or higher:

```
aws --version
```

### Generate Access Key ID and Secret Access Key
You will need an access key ID and secret access key to interact with AWS services.

1. Use your AWS account email address and password to sign in to the AWS Management Console.

2. Open the IAM Dashboard page,

3. In the navigation pane of the console, choose ***Users***.

4. Choose your IAM user name (not the check box).

5. Choose the ***Security credentials*** tab and then choose ***Create access key***.

6. To see the new access key, choose ***Show***. Your credentials will look something like this:

  ```
  Access key ID: AKIAIOSFODNN7EXAMPLE
  Secret access key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  ```

7. To download the key pair, choose Download .csv file. Store the keys in a secure location.

*Keep the keys confidential in order to protect your AWS account, and never email them. Do not share them outside your organization, even if an inquiry appears to come from AWS or Amazon.com. No one who legitimately represents Amazon will ever ask you for your secret key.*

### Configure the AWS CLI
`aws configure` command is the fastest way to set up your AWS CLI installation.

You will be prompted to provide four pieces of information - AWS Access Key ID, AWS Secret Access Key, Default region name, and Default output format

Use the Access Key and Secret Access Key generated from the previous step. In this tutorial we will be using `us-east-1` as the default region and `json` as the default output format.

> example:

```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json
```

### Create a Key Pair
AWS uses public-key cryptography to secure the login information for your instance. A Linux instance in AWS has no password; you need to use a key pair to log in to your instance securely.

Generate a key pair that will be used to access the compute instances.

```
aws ec2 create-key-pair \
  --key-name kubernetes-the-hard-way \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/kubernetes-the-hard-way.pem
```

```
chmod 400 ~/.ssh/kubernetes-the-hard-way.pem
```

### Helper Functions
In order to improve readability of this tutorial, you will need to define the following shell functions.

```
function hostname_from_instance() {
    echo $(aws ec2 describe-instances --filters "{\"Name\":\"tag:Name\", \"Values\":[\"$1\"]}" "{\"Name\":\"instance-state-name\", \"Values\":[\"running\"]}"  --query='Reservations[0].Instances[0].PublicDnsName' | tr -d '"')
}

function ip_from_instance() {
    echo $(aws ec2 describe-instances --filters "{\"Name\":\"tag:Name\", \"Values\":[\"$1\"]}" "{\"Name\":\"instance-state-name\", \"Values\":[\"running\"]}"  --query='Reservations[0].Instances[0].PublicIpAddress' | tr -d '"')
}

function ssh-aws() {
    ssh -i ~/.ssh/kubernetes-the-hard-way.pem ubuntu@$(ip_from_instance "$1") ${@:2}
}
```

> Use the `aws ec2 describe-regions` command to view additional regions and `aws ec2 describe-availability-zones --region region-name` to view additional zones.

Next: [Installing the Client Tools](02-client-tools.md)
