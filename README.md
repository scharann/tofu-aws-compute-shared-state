# OpenTofu AWS Compute (Shared State Collision Lab)

## Purpose & Problem Being Solved
This repository provisions an EC2 instance and security group in `ap-southeast-2` while reading network IDs via `data.terraform_remote_state.network` from the **exact same S3 state key** (`shared-collision/terraform.tfstate`) that it writes its own state back to.

**The Problem Demonstrated:**
This setup proves why every root module requires an isolated `.tfstate` key. When a downstream module reads `terraform_remote_state` from the same file it overwrites during `tofu apply`, it triggers both a live infrastructure deletion race and **circular state self-destruction** (erasing the very outputs it needs for future runs).

## Prerequisites
1. **Remote State Backend:** The S3 bucket (`<YOUR-S3-BUCKET-NAME>`) and DynamoDB lock table (`tofu-lab-locks`) must exist in `ap-southeast-2` (see `tofu-aws-network-shared-state` README for creation commands).
2. **Network Stack Deployed:** `tofu-aws-network-shared-state` must be applied first so `shared-collision/terraform.tfstate` exists in S3 with initial network resources and outputs.

## How It Works (The Collision Sequence)
1. **Before `apply`:** `shared-collision/terraform.tfstate` holds the 8 network resources and outputs (`vpc_id`, `public_subnet_ids`) created by `tofu-aws-network-shared-state`.
2. **During `tofu plan`:** `data.terraform_remote_state.network` reads those outputs successfully, and OpenTofu plans `2 to add, 0 to change, 8 to destroy` (because the 8 network resources are not defined in this repo's `.tf` files).
3. **During `tofu apply`:** OpenTofu deletes the public subnets (`0s`) and creates `aws_security_group.ec2_sg` (`3s`) inside the VPC—causing `aws_instance.web` to fail (missing subnet) and `aws_vpc.main` to hang in a `DependencyViolation` retry loop (blocked by the new security group).
4. **End of `tofu apply`:** OpenTofu writes this repo's state back to `shared-collision/terraform.tfstate`, erasing all `output` blocks (`vpc_id`, `public_subnet_ids`).

## Expected Errors
1. **During `tofu apply`:**
   - `api error InvalidSubnetID.NotFound: The subnet ID 'subnet-xxxxxxxx' does not exist` (on `aws_instance.web`, because the subnet was destroyed at `0s` before EC2 creation started).
   - `api error DependencyViolation: The vpc 'vpc-xxxxxxxx' has dependencies and cannot be deleted.` (on `aws_vpc.main`, looping until timeout or `Ctrl+C`).
2. **On any subsequent `tofu plan` or `tofu destroy`:**
~~~text
Error: Unsupported attribute

  on main.tf line 4, in resource "aws_security_group" "ec2_sg":
   4:   vpc_id = data.terraform_remote_state.network.outputs.vpc_id

This object does not have an attribute named "vpc_id".
~~~

## How to Clean Up & Verify
Because `tofu destroy` crashes with `Unsupported attribute` once the state outputs are wiped out, remove the stranded resources manually:
1. **Check what survived in the corrupted state file:**
~~~bash
tofu state list
~~~
*(Expected: `aws_security_group.ec2_sg` and `aws_vpc.main` remain tracked together).*
2. **Detach both resources from OpenTofu state:**
~~~bash
tofu state rm aws_security_group.ec2_sg aws_vpc.main
~~~
3. **Delete both resources in AWS (`ap-southeast-2`):**
~~~bash
aws ec2 delete-security-group --group-id <SG-ID> --region ap-southeast-2
aws ec2 delete-vpc --vpc-id <VPC-ID> --region ap-southeast-2
~~~
4. **Verify Everything Is Clean:**
   - `tofu state list` shows zero managed resources.
   - `aws ec2 describe-security-groups --region ap-southeast-2` and `aws ec2 describe-vpcs --region ap-southeast-2` show no lab resources remaining.
