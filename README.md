# AWS Infrastructure Automation with Ansible

This repository contains Ansible playbooks to automate AWS infrastructure provisioning and management. The solution provides a streamlined approach to launch and terminate AWS EC2 instances with appropriate security groups and SSH access.

## Overview

The repository contains a collection of Ansible playbooks and templates that facilitate:

- Provisioning of AWS EC2 instances with proper security groups
- SSH key pair management
- Dynamic inventory generation
- Clean infrastructure removal

## Prerequisites

- Ansible 2.9 or higher
- `amazon.aws` collection installed
- AWS IAM credentials with appropriate permissions
- Python boto3 library

## Installation

1. Clone this repository
2. Install required Ansible collections:
   ```
   ansible-galaxy collection install amazon.aws
   ```
3. Install Python dependencies:
   ```
   pip install boto3
   ```

## AWS IAM Configuration

It is recommended to create a dedicated IAM user with the minimum required permissions for these playbooks to operate securely.

### Creating an IAM User with Restricted Policy

1. Sign in to the AWS Management Console and navigate to the IAM service
2. Choose "Users" from the sidebar and click "Add user"
3. Enter a username (e.g., `ansible-automation`) and select "Programmatic access"
4. On the permissions page, select "Attach policies directly"
5. Click "Create policy," go to the JSON tab, select the appropriate policy from the list below, and paste it into the editor:

<details>
<summary><strong>EC2 Automation Policy without Instance type limitations</strong></summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EC2KeyPairOperations",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateKeyPair",
                "ec2:DeleteKeyPair",
                "ec2:DescribeKeyPairs"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2SecurityGroupOperations",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateSecurityGroup",
                "ec2:DeleteSecurityGroup",
                "ec2:DescribeSecurityGroups",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupIngress"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2InstanceOperations",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:TerminateInstances",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceStatus",
                "ec2:DescribeInstanceAttribute",
                "ec2:ModifyInstanceAttribute",
                "ec2:CreateTags"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2AMIAccess",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeImages",
                "ec2:DescribeVpcs",
                "ec2:DescribeSubnets"
            ],
            "Resource": "*"
        },
        {
            "Sid": "TaggingPermissions",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags",
                "ec2:DescribeTags"
            ],
            "Resource": "*"
        }
    ]
}
```
</details>

<details>
<summary><strong>EC2 Automation Policy with limitations to t2.micro Instance type</strong></summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EC2KeyPairOperations",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateKeyPair",
                "ec2:DeleteKeyPair",
                "ec2:DescribeKeyPairs"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2SecurityGroupOperations",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateSecurityGroup",
                "ec2:DeleteSecurityGroup",
                "ec2:DescribeSecurityGroups",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupIngress"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2InstanceOperations",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:instance/*"
            ],
            "Condition": {
                "StringEquals": {
                    "ec2:InstanceType": "t2.micro"
                }
            }
        },
        {
            "Sid": "EC2RunInstancesAdditionalResources",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:volume/*",
                "arn:aws:ec2:*:*:network-interface/*",
                "arn:aws:ec2:*:*:key-pair/*",
                "arn:aws:ec2:*:*:security-group/*",
                "arn:aws:ec2:*::image/*",
                "arn:aws:ec2:*:*:subnet/*"
            ]
        },
        {
            "Sid": "EC2ManageInstances",
            "Effect": "Allow",
            "Action": [
                "ec2:TerminateInstances",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceStatus",
                "ec2:DescribeInstanceAttribute",
                "ec2:ModifyInstanceAttribute",
                "ec2:CreateTags"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2AMIAccess",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeImages",
                "ec2:DescribeVpcs",
                "ec2:DescribeSubnets"
            ],
            "Resource": "*"
        },
        {
            "Sid": "TaggingPermissions",
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags",
                "ec2:DescribeTags"
            ],
            "Resource": "*"
        }
    ]
}
```
</details>

6. Name the policy (e.g., `AnsibleEC2AutomationPolicy`), add a description, and create the policy
7. Return to the user creation workflow, refresh the policy list, and attach your new policy
8. Complete the user creation process

### Obtaining Access Keys

After the user is created, you will be provided with an Access Key ID and Secret Access Key. These credentials should be securely stored in your `secrets.yml` file:

1. Copy the Access Key ID and Secret Access Key from the success page (or download the CSV file)
2. Update your `secrets.yml` file:
   ```yaml
   access_key: YOUR_ACCESS_KEY_ID
   secret_key: YOUR_SECRET_ACCESS_KEY
   ```
3. Secure the file using `ansible-vault`:
   ```
   ansible-vault encrypt secrets.yml
   ```

**Important**: The access key and secret key will only be displayed once. If you don't save them, you'll need to generate a new access key pair.

## Configuration

Update the following configuration files according to your environment:

### infos.yml

Contains general configuration parameters:
```yaml
region: eu-central-1             # AWS region
project: wp_stack                # Project name
env: test                        # Environment name
instances_type: t2.micro         # EC2 instance type
instances_running: 2             # Number of instances to provision
ami_id: ami-0b74f796d330ab49c    # AMI ID for your region
```

### secrets.yml

Contains AWS credentials:
```yaml
access_key: YOUR_ACCESS_KEY
secret_key: YOUR_SECRET_KEY
```

**Note:** It is recommended to encrypt this file using `ansible-vault`:
```
ansible-vault encrypt secrets.yml
```

## Usage

### Provisioning Infrastructure

To provision the infrastructure:

```
ansible-playbook provision.yml
```

This will:
1. Create an SSH key pair
2. Create a security group with SSH and HTTP access
3. Launch the specified number of EC2 instances
4. Generate a dynamic inventory file for further management

### Cleaning Infrastructure

To remove all provisioned resources:

```
ansible-playbook clean.yml
```

This will:
1. Terminate all EC2 instances associated with the project and environment
2. Delete the security group
3. Remove the key pair from AWS
4. Remove the key from control plane

## Project Structure

- `provision.yml` - Main playbook for infrastructure provisioning
- `clean.yml` - Playbook for infrastructure removal
- `infos.yml` - Environment configuration
- `secrets.yml` - AWS credentials
- `inventory.ini.j2` - Template for generating dynamic inventory
- `[project]-[env].pem` - Generated SSH private key
- `aws-[project]-[env].ini` - Generated inventory file

## Inventory Management

The playbook automatically generates an inventory file (`aws-[project]-[env].ini`) with all provisioned instances for easy management. Each instance is named as `node[index]` with appropriate connection parameters.

## Security Considerations

- The playbooks create a security group that allows SSH (port 22) and HTTP (port 80) access from any IP (0.0.0.0/0). Consider restricting this to specific IPs in production environments.
- AWS credentials are stored in the `secrets.yml` file. Ensure this file is properly secured.
- SSH private keys are generated with `0400` permissions (read-only for the owner).
- Use the restricted IAM policy provided to limit the scope of permissions.

