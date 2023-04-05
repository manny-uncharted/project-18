# AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 3 – REFACTORING

## Table of Contents
- [Introduction](#introduction)
- [prerequisites](#prerequisites)
- [Introducing Backend on S3](#introducing-backend-on-s3)
- [Refactoring](#refactoring)
    - [ALB module](#alb-module)
    - [VPC module](#vpc-module)
    - [RDS module](#rds-module)
    - [Autoscaling Group module](#autoscaling-group-module)
    - [Security Group module](#security-group-module)
    - [EFS module](#efs-module)
    -[Compute module](#compute-module)
    - [Defining Modules in Root Main.tf](#defining-modules-in-root-maintf)








## Introduction
From our previous project, we have developed a Terraform project that can implement an AWS infrastructure code using Terraform. In this project, we will work on refactoring the code.

## Prerequisites
- [Terraform](https://www.terraform.io/downloads.html)
- [AWS Account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [AWS IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)
- [AWS IAM User Access Key](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
- [AWS S3 Bucket](https://s3.console.aws.amazon.com/s3/)
- Code Editor (VS Code, Atom, Sublime Text, etc.)
- [Past Project Files](https://github.com/manny-uncharted/project-17)


## Introducing Backend on S3
In the previous project, we have used the local backend to store the state file. This is not a good practice as it is not recommended to store the state file locally. In this project, we will be using the S3 backend to store the state file. This will allow us to share the state file with other team members and also allow us to collaborate on the project.

- Create a file and name it `backend.tf` and add the following code:
```terraform
resource "aws_s3_bucket" "terraform_state" {
  bucket = "<your-name>-dev-terraform-bucket"
  # Enable versioning so we can see the full revision history of our state files
  versioning {
    enabled = true
  }
  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
```
Note: Terraform stores secret data inside state files. Passwords and secret keys processed by resources are always stored in there. There we need to always enable encryption. This is achieved with the [server_side_encryption_configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html)

result:

![Backends init s3 bucket](img/backends-init-s3-bucket.png)

- Create a DynamoDB table to handle locks and perform consistency checks. In our previous projects, locks were handled with a local file shown in `terraform.tfstate.lock.info`. Since we want to enable collaboration, which made us to configure S3 as our backend to store our state file, we need to do the same to handle locking. Therefore, with a cloud database like DynamoDB, anyone running Terraform against the same infrastructure can use a central location to control a situation where Terraform is running at the same time from multiple individuals on our DevOps Team.
Add the following code to our `backend.tf` file for DynamoDB resource for locking and consistency checking:
```terraform
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }

  tags = merge(
    var.tags,
    {
      Name = format("%s-DynamoDB-Terraform-State-Lock-Table-%s", var.name, var.environment)
    },
  )
}
```

result:

![Backends init dynamodb table](img/backends-init-dynamodb-table.png)

- Now let's run `terraform init` and `terraform apply` to create the S3 bucket and DynamoDB table. This will be used to store the state file and handle locking and consistency checks. As Terraform expects us to have the S3 bucket and DynamoDB created before we can use them as our backend, we need to run `terraform apply` to create them. After running `terraform apply`, we will get the following output:
```bash
terraform init
terraform apply --auto-approve
```

result:

![Backends init terraform apply](img/backends-init-terraform-apply.png)

- Now let's configure our backend to use the S3 bucket and DynamoDB table we just created. Add the following code to our `backend.tf` file:
```terraform
terraform {
  backend "s3" {
    bucket         = "manny-dev-terraform-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

result:

![Backends init backend.tf](img/backends-init-backend-tf.png)

- Now it's time to run `terraform init` again to initialize the backend. After running `terraform init`, we will get the following output:
```bash
terraform init
```

result:

![Backends init terraform init](img/backends-init-terraform-init.png)

- Now let's verify the changes. Open your AWS console and navigate to S3. You will see the S3 bucket we created earlier. Click on the bucket and you will see the state file we created earlier. This is the state file we will be using to manage our infrastructure. This is the state file we will be sharing with our team members and collaborators.

result:

![Backends init s3 bucket state file](img/backends-init-s3-bucket-state-file.png)

- Let's also check on our Dynamo DB table. Navigate to the DynamoDB table inside AWS and leave the page open in your browser. Run terraform plan and while that is running, refresh the browser and see how the lock is being handled:

result:

![Backends init dynamodb table lock](img/backends-init-dynamodb-table-lock.png)

- Now let's re-initialize the backend. Run `terraform init` again to initialize the backend. After running `terraform init`, we will get the following output:
```bash
terraform init
```

result:

![Backends init terraform init](img/backends-init-terraform-init.png)

- Let's verify the changes that happened after re-initializing our backend. We would open our S3 bucket and also our DynamoDB table
    - Open your AWS console and navigate to S3. You will see the S3 bucket we created earlier. Click on the bucket and you will see the state file we created earlier. This is the state file we will be using to manage our infrastructure. This is the state file we will be sharing with our team members and collaborators.

    result:

    ![Backends init s3 bucket state file](img/backends-init-s3-bucket-state-file.png)

    - Let's also check on our Dynamo DB table. Navigate to the DynamoDB table inside AWS and leave the page open in your browser. Run terraform plan and while that is running, refresh the browser and see how the lock is being handled:

    result:

    ![Backends init dynamodb table lock](img/backends-init-dynamodb-table-lock.png)


Before we apply our current changes let's add an output such that the S3 bucket `Amazon Resource Name (ARN)` is displayed after running `terraform apply`. This will allow us to use the S3 bucket ARN in other projects.

- Now let's create a new file and name it `output.tf` and add the following code:
output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
  description = "The ARN of the S3 bucket"
}
output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "The name of the DynamoDB table"
}

result:

![Backends init output.tf](img/backends-init-output-tf.png)

- Now that we have everything ready to go. Let's run `terraform apply` to apply our changes. After running `terraform apply`, we will get the following output:
```bash
terraform apply --auto-approve
```

result:

![Backends init terraform apply](img/backends-terraform-apply.png)

With help of the remote backend and locking configuration that we have just configured, collaboration is no longer a problem.


## Refactoring
In our previous project, we created the resources for our architecture in a single directory, with this approach things can get messy pretty fast and when we need to modify our code and add or delete resources it makes it difficult to find. In this section, we will refactor our code, by separating them into terraform modules to make it more organized and easier to manage.

- Let's create a new directory and name it `modules`. This will be the directory where we will store our Terraform modules.
```bash
mkdir modules
```

- In our modules directory, let's create the following directories:
```bash
cd modules
mkdir ALB Autoscaling EFS RDS compute VPC Security
```

result:

![modules directory](img/modules-directory.png)

- Let's move all the files that have to do with networking to the VPC directory.

result:

![modules VPC directory](img/modules-vpc-directory.png)

- Let's also move the `rds.tf` file to the RDS directory.

result:

![modules RDS directory](img/modules-rds-directory.png)

- Let's move the `alb.tf`, `cert.tf` and `outputs.tf` files to the ALB directory.

result:

![modules ALB directory](img/modules-alb-directory.png)

- Let's move the `efs.tf` file to the EFS directory.

result:

![modules EFS directory](img/modules-efs-directory.png)

- Let's move the `asg-*.tf`, `*sh`,  files to the Autoscaling directory.

result:

![modules Autoscaling directory](img/modules-autoscaling-directory.png)

- Let's move the `security.tf` file to the Security directory.

result:

![modules Security directory](img/modules-security-directory.png)

- Now in let's create the following files in the root directory:
    - `main.tf`
    - `variables.tf`
    - `outputs.tf`
    - `providers.tf`

result:

![modules root directory](img/modules-root-directory.png)


- In our root directory `providers.tf` add the following lines of code:
```terraform
provider "aws" {
    region = var.region
}
```
Note: Ensure you remove this from the `main.tf` file present in our VPC directory.

result:

![modules providers.tf](img/modules-providers-tf.png)

- In every module directory, we would need to create the following files:
    - `main.tf`
    - `variables.tf`
    - `outputs.tf`


### ALB module
- Now beginning with the `ALB module`, let's create the `main.tf` file and move the code in the `alb.tf` file to the `main.tf` file.

result:

![modules ALB main.tf](img/modules-alb-main-tf.png)

- We would also need to declare variables for every hard-coded value in our `main.tf` file. Let's create the `variables.tf` file and add the following code:
```terraform
### ------ ALB/variables.tf ------ ###

# The security froup for external loadbalancer
variable "public-sg" {
  description = "Security group for external load balancer"
}

# The public subnet froup for external loadbalancer
variable "public-sbn-1" {
  description = "Public subnets to deploy external ALB"
}

variable "public-sbn-2" {
  description = "Public subnets to deploy external  ALB"
}

variable "vpc_id" {
  type        = string
  description = "The vpc ID"
}

variable "private-sg" {
  description = "Security group for Internal Load Balance"
}

variable "private-sbn-1" {
  description = "Private subnets to deploy Internal ALB"
}
variable "private-sbn-2" {
  description = "Private subnets to deploy Internal ALB"
}

variable "ip_address_type" {
  type        = string
  description = "IP address for the ALB"

}

variable "load_balancer_type" {
  type        = string
  description = "te type of Load Balancer"
}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}

variable "name" {
    type = string
    description = "name of the loadbalancer"
    default = "ext-alb"
  
}

variable "environment" {
  type        = string
  description = "The environment name"
  default = "dev"
}

variable "port" {
  type        = number
  description = "The port number"
  default = 443
}

variable "protocol" {
  type        = string
  description = "The protocol type"
  default = "HTTPS"
}

variable "company_name" {
  type        = string
  description = "The company name"
}

variable "interval" {
  type        = number
  description = "The interval for health check"
  default = 10
}

variable "path" {
  type        = string
  description = "The path for health check"
  default = "/healthstatus"
}

variable "timeout" {
  type        = number
  description = "The timeout for health check"
  default = 5
}

variable "healthy_threshold" {
  type        = number
  description = "The healthy threshold for health check"
  default = 5
}

variable "unhealthy_threshold" {
  type        = number
  description = "The unhealthy threshold for health check"
  default = 2
}

variable target_type {
  type        = string
  description = "The target type"
  default = "instance"
}

variable "default_action_type" {
  type        = string
  description = "The default action type"
  default = "forward"
}

variable "priority" {
  type        = number
  description = "The priority for the rule"
  default = 99
}
```

result:
![modules ALB variables.tf](img/modules-alb-variable-tf.png)

- Let's now assign the variable we declared in the `variables.tf` file to the hard-coded values in the `main.tf` file. The `main.tf` file should look like this:
```terraform
### ----- ALB/main.tf ------ ###
# Create external application load balancer for reverse proxy nginx.
resource "aws_lb" "ext-alb" {
  name     = var.name
  internal = false
  security_groups = [
    var.public-sg
  ]

  subnets = [
    var.public-sbn-1,
    var.public-sbn-2
  ]

  tags = merge(
    var.tags,
    {
      Name = format("%s-ext-alb-%s", var.company_name, var.environment)
    },
  )

  ip_address_type    = var.ip_address_type
  load_balancer_type = var.load_balancer_type
}

# Create target group to point to its targets.
resource "aws_lb_target_group" "nginx-tg" {
  health_check {
    interval            = var.interval # 10
    path                = var.path # "/healthstatus"
    protocol            = var.protocol # "HTTPS"
    timeout             = var.timeout # 5
    healthy_threshold   = var.healthy_threshold # 5
    unhealthy_threshold = var.unhealthy_threshold # 2
  }
  name        = format("%s-nginx-tg-%s", var.company_name, var.environment)
  port        = var.port # 443
  protocol    = var.protocol # "HTTPS"
  target_type = var.target_type # "instance"
  vpc_id      = var.vpc_id
}

# Create listener to redirect traffic to the target group.
resource "aws_lb_listener" "nginx-listener" {
  load_balancer_arn = aws_lb.ext-alb.arn
  port              = var.port # 443
  protocol          = var.protocol # "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.teskers.certificate_arn

  default_action {
    type             = var.default_action_type # "forward"
    target_group_arn = aws_lb_target_group.nginx-tg.arn
  }
}

#Internal Load Balancers for webservers
resource "aws_lb" "int-alb" {
  name     = "int-alb"
  internal = true
  security_groups = [
    var.private-sg
  ]

  subnets = [
    var.private-sbn-1,
    var.private-sbn-2
  ]

  tags = merge(
    var.tags,
    {
      Name = format("%s-int-alb-%s", var.company_name, var.environment)
    },
  )

  ip_address_type    = var.ip_address_type # "ipv4"
  load_balancer_type = var.load_balancer_type # "application"
}

# --- target group  for wordpress -------
resource "aws_lb_target_group" "wordpress-tg" {
  health_check {
    interval            = var.interval # 10
    path                = var.path # "/healthstatus"
    protocol            = var.protocol # "HTTPS"
    timeout             = var.timeout # 5
    healthy_threshold   = var.healthy_threshold # 5
    unhealthy_threshold = var.unhealthy_threshold # 2
  }

  name        = format("%s-wordpress-tg-%s", var.company_name, var.environment)
  port        = var.port # 443
  protocol    = var.protocol # "HTTPS"
  target_type = var.target_type # "instance"
  vpc_id      = var.vpc_id
}

# --- target group for tooling -------
resource "aws_lb_target_group" "tooling-tg" {
  health_check {
    interval            = var.interval # 10
    path                = var.path # "/healthstatus"
    protocol            = var.protocol # "HTTPS"
    timeout             = var. timeout # 5
    healthy_threshold   = var.healthy_threshold # 5
    unhealthy_threshold = var.unhealthy_threshold # 2
  }

  name        = format("%s-tooling-tg-%s", var.company_name, var.environment)
  port        = var.port # 443
  protocol    = var.protocol # "HTTPS"
  target_type = var.target_type # "instance"
  vpc_id      = var.vpc_id
}

# For this aspect a single listener was created for the wordpress which is default,
# A rule was created to route traffic to tooling when the host header changes
resource "aws_lb_listener" "web-listener" {
  load_balancer_arn = aws_lb.int-alb.arn
  port              = var.port # 443
  protocol          = var.protocol # "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.teskers.certificate_arn

  default_action {
    type             = var.default_action_type # "forward"
    target_group_arn = aws_lb_target_group.wordpress-tg.arn
  }
}

# listener rule for tooling target
resource "aws_lb_listener_rule" "tooling-listener" {
  listener_arn = aws_lb_listener.web-listener.arn
  priority     = var.priority # 99

  action {
    type             = var.default_action_type # "forward"
    target_group_arn = aws_lb_target_group.tooling-tg.arn
  }

  condition {
    host_header {
      values = ["tooling.teskers.online"]
    }
  }
}
```

result:
![modules ALB main.tf](img/modules-alb-main-tf-variable.png)

- Let's now create the `outputs.tf` file. The `outputs.tf` file should look like this:
```terraform
### -------- ALB/output.tf -------- ###

output "alb_dns_name" {
  value = aws_lb.ext-alb.dns_name
}

output "alb_target_group_arn" {
  value = aws_lb_target_group.nginx-tg.arn
}

output "wordpress-tg" {
  description = "The target group for wordpress"
  value = aws_lb_target_group.wordpress-tg.arn
}

output "tooling-tg" {
  description = "The target group for tooling"
  value = aws_lb_target_group.tooling-tg.arn
}

output "nginx-tg" {
  description = "The target group for nginx"
  value = aws_lb_target_group.nginx-tg.arn
}
```

result:
![modules ALB output.tf](img/modules-alb-output-tf.png)


### VPC Module

As we did in the ALB module, we'll create a VPC module. The VPC module will create the VPC, subnets, route tables, internet gateway, NAT gateway, and the security groups. The VPC module will be created in the `VPC` directory. The `VPC` directory will have the following files:

- In refactoring the `main.tf` file would look like this:
```terraform
### ------ VPC/main.tf ------ ###

resource "random_integer" "random" {
  min = 1
  max = 10
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

  tags = merge(
    var.tags,
    {
      Name = format("%s-VPC-%s", var.name, var.environment)
    },
  )
}

# Get list of availablility zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Create public subnets1
resource "aws_subnet" "public" {
  count                   = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index] # cidrsubnet(var.vpc_cidr, 8, count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Public-subnet-%s", var.name, count.index + 1)
    },
  )
}

# Create private subnet
resource "aws_subnet" "private" {
  count                   = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.private_subnets[count.index] # cidrsubnet(var.vpc_cidr, 8, count.index + 2)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-subnet-%s", var.name, count.index + 1)
    },
  )
}
```

- The `variables.tf` file would look like this:
```terraform
### ------ VPC/variables.tf ------ ###
variable "region" {
  default = "us-east-1"
}

variable "vpc_cidr" {
  default = "172.16.0.0/16"
}

variable "enable_dns_support" {
  default = "true"
}

variable "enable_dns_hostnames" {
  default = "true"
}

variable "enable_classiclink" {
  default = "false"
}

variable "enable_classiclink_dns_support" {
  default = "false"
}

variable "preferred_number_of_public_subnets" {
  type        = number
  description = "Number of public subnets to create. If not specified, all available AZs will be used."
}

variable "preferred_number_of_private_subnets" {
  # default = 4
  type        = number
  description = "Number of private subnets to create. If not specified, all available AZs will be used."
}

variable "private_subnets" {
  type        = list(any)
  description = "List of private subnets"
}

variable "public_subnets" {
  type        = list(any)
  description = "list of public subnets"

}

variable "name" {
  description = "Name to be used on all the resources as identifier."
  type        = string
  default     = "ACS"
}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}

variable "environment" {
  description = "Environment where resources are being created."
  type        = string
  default     = "dev"
}

variable "destination_cidr_block" {
  description = "The CIDR block to allow access to the VPC."
  type        = string
  default     = "0.0.0.0/0"
}
```

- The `outputs.tf` file would look like this:
```terraform
output "public_subnets-1" {
  value       = aws_subnet.public[0].id
  description = "The first public subnet in the subnets"
}

output "public_subnets-2" {
  value       = aws_subnet.public[1].id
  description = "The first public subnet"
}


output "private_subnets-1" {
  value       = aws_subnet.private[0].id
  description = "The first private subnet"
}

output "private_subnets-2" {
  value       = aws_subnet.private[1].id
  description = "The second private subnet"
}


output "private_subnets-3" {
  value       = aws_subnet.private[2].id
  description = "The third private subnet"
}


output "private_subnets-4" {
  value       = aws_subnet.private[3].id
  description = "The fourth private subnet"
}


output "vpc_id" {
  value = aws_vpc.main.id
}


output "instance_profile" {
  value = aws_iam_instance_profile.ip.id
}
```

Note: The `outputs.tf` file is used to output the values of the resources created in the module. This is useful when we want to use the output of one module as input to another module.   


### RDS module


### Autoscaling Group module

### Security Group module

### EFS module

### Compute module

### Defining Modules in Root Main.tf
In this section all the modules we've created earlier have to be referenced in the `main.tf` file in the root directory. There is a structure to this:
    - First, we have to call the module. The module name should be the same as the directory name.
    - Second, we have the `source` attribute which is the path to the module directory.
    - Third, all the variables declared in the module should be defined in the module block.

- The `main.tf` file would look like this:
```terraform
### ------ main.tf ------ ###
module "<module-name-corresponding-to-folder-name>" {
    source = "./<path-to-module>"
    <variable-name> = <variable-value>
}
```

