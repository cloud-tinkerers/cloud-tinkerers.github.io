---
title: "Terraform: Quick Dive #1: Challenge solutions"
date: 2024-08-21T07:07:07+01:00
language: en
featured_image: ../assets/images/posts/terraform-quick-dive-1-solutions.jpg
summary: Discover how to get started with Terraform in AWS.
description: Discover how to get started with Terraform in AWS.
author: Conor
authorimage: ../assets/images/global/icon.webp
categories: cloud, devops
tags: cloud, aws, infrastructure, infrastructure as code
---
In [part one](https://cloudtinkerers.com/posts/terraform-quick-dive-%231/) I posed a handful of challenges to help you develop your Terraform skills further. I encourage you to have a go at them yourself first, that exploration and struggle is where true learning happens. But I am going to solve the challenges in this post.

You can find the updated code for these challenges here: https://github.com/cloud-tinkerers/terraform-projects/tree/terraform-quick-dive-1-solutions

----------

# Challenge #1

*Figure out how to get access to your server via SSH or AWS Session Manager.*

## SSH

In part 1 we created a security group that allowed access to our server on port 80. Allowing the minimum level of access possible to serve the needs of your application is a good practice to stick by, but if you know anything about SSH then you know that we will need to open up another port to our server to enable it.

`sg.tf``

```terraform
resource "aws_vpc_security_group_ingress_rule" "allow_ssh" {
  security_group_id = aws_security_group.main.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 22
  ip_protocol       = "tcp"
  to_port           = 22
}
```

By default SSH uses port 22, so we need to add a new resource that defines another ingress rule to allow traffic on port 22 to make it to our server. This rule applies to all IPv4 traffic (0.0.0.0/0 means 'any'). I wouldn't recommend this setup for a production environment, but we'll be okay for now.

Next we need a way to make our server aware of the SSH key we'll be using to access it, fortunately Terraform makes this pretty easy.

`ec2.tf`

```terraform
resource "aws_key_pair" "ssh_key" {
  name = "main"
  public_key = "<your-PUBLIC-ssh-key>"
}
```

Creating an SSH key is out of scope for this tutorial, but on a Linux system you can find your public SSH key by running `cat ~/.ssh/*.pub` - copy the output manually, or if you have xclip you can use `cat ~/.ssh/*.pub | xclip -sel clip` - then paste it inside the quotes of this resource.

Finally, add the following argument to your aws_instance resource:

```terraform
  key_name                    = aws_key_pair.ssh_key.id
```

Applying this Terraform should now allow you to SSH to the public IP of your server using the ec2-user:

![ssh-output](/images/ssh-output.png)

## Session Manager

AWS Systems Manager (SSM) has a great feature called Session Manager which allows you to access your instance without opening up any extra ports. Your instance has to be managed by SSM for this work. This is how you set that up.

```terraform
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

data "aws_iam_policy" "ssm_managed_instance_core" {
  name= "AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy_attachment" "ssm_role_attachment" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = data.aws_iam_policy.ssm_managed_instance_core.arn
}

resource "aws_iam_instance_profile" "ec2_role" {
  name = "ec2-role"
  role = aws_iam_role.ec2_role.name
}
```
There's a few parts to this and if you're not familiar with AWS IAM it will seem confusing. First we have to create a role that the EC2 service has permissions to assume. This forms a trust relationship between the instance and the role, allowing it to make us of the policies attached to it.

There's a data source for the AWS managed "AmazonSSMManagedInstanceCore" policy, we can then use a policy attachment resource to attach this policy to the role.

Finally, an IAM instance profile is created. This is what we'll attach to the EC2 instance. Add this line to your aws_instance resource:

```terraform
iam_instance_profile        = aws_iam_instance_profile.ec2_role.name
```

Once you've applied this you should be able to connect to your instance using the following:

```bash
aws ssm start-session --target {instance-id}
```
--------

# Challenge #2

*Define a variable in your code, and use that variable to name your server.*

For this we're going to create two files, variables.tf and terraform.tfvars. In the former we need to define a variable.

```terraform
variable "project" {
  type = string
  description = "The current project."
}
```

In terraform.tfvars we'll create a value for this variable.

```terraform
project = "terraform-project"
```

Finally, in order to pass this variable to our EC2 instance as a name we'll update the tag:

```terraform
  tags = {
    Name = "${var.project}"
  }
}
```

When you apply this your instance should have the project name:

![variable](/images/variable.png)

# Challenge #3

*Investigate the terraform.tfstate file and learn more about what Terraform state is.*

State is an important concept to understand in Terraform. Other tools, such as Ansible, are capable of creating infrastructure, but largely inferior in that many of them do not track state. But what does that mean?

If I tell Ansible that I want 1 server it will create it. Later down the line I may realise that the needs of my application are greater, logic suggests that I update my Ansible to 2 servers. Except now I'll have 3 servers. Why? Because Ansible didn't keep track of the first server after creating it, so when I went back to change it to 2, Ansible simply created 2 more servers.

By contrast, Terraform keeps track of state. If I tell Terraform I want 1 server, and later revise it to 2, I will have 2 servers. This is important. Everything created and maintained within your Terraform repo is tracked. As we work you may have noticed a file appear called terraform.tfstate. We can have a look inside this.
```terraform
"mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "ami": "ami-04e49d62cf88738f1",
            "arn": "arn:aws:ec2:eu-west-1:471211282739:instance/i-00515e7c8e03f0d11",
            "associate_public_ip_address": true,
            "availability_zone": "eu-west-1a",
```

This excerpt shows a few of the details that have been tracked for our aws_instance resource. It tells us the Terraform name for the resource, the AMI that was used to create it, the ARN/InstanceID, the fact that it has a public IP address and the availability zone that it was placed in.

If you were to destroy your infrastructure through Terraform, this would disappear. *However*, if you were to terminate the instance through AWS, this would not be recorded in the state because it happened outside of Terraform. You have now created drift. The next time you you apply your Terraform it will compare your Terraform configuration against the state, and then also against the infrastructure. It will tell you that it wants to create this instance because it has detected that it no longer exists.

It is important to note that 'local state' (what we're currently using) is a bad practise in work environments because your colleagues who may also be working on your infrastructure will not have access to your local state file. Terraform offers 'remote state' to solve this problem, and can be configured to use an S3 bucket along with various other options.

You can read more about state here: https://developer.hashicorp.com/terraform/language/state

---

# Challenge #4

*Add some tags to your instance and figure out how to get the EC2 instance to read this metadata and display it in your HTML page.*

As you develop more and more things in the cloud you begin to realise that tags are invaluable part of your toolkit, and with the right approach to them you can more easily automate your infrastructure.

This is a tricky one to set up, and if you've followed my guide so far your instance will default to requiring IMDSv2. This is better as it requires you to pass along a valid session token with your metadata requests, but there's a bit more involved in setting it up.

You can review the IMDS documentation here: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html

Inside the `aws_instance` resource we need to add some metadata options.

```terraform
  metadata_options {
    http_endpoint               = "enabled"
    instance_metadata_tags      = "enabled"
    http_put_response_hop_limit = 2
    http_tokens                 = "required"
  }
```

You can read more about these [here](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/instance#metadata_options). In essence we are enabling the metadata service, enforcing tokens in order to access it, and allowing the instance to read its own tags.

With that in place we can update our bash startup script to do something with the instance tags.

```bash
#!/bin/bash

TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
project=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Name)
sudo yum upgrade -y
sudo yum install -y httpd
echo "Welcome to $${project}" > /var/www/html/index.html
sudo systemctl start httpd
EOF
```

We've created a `TOKEN` variable to store the output of our curl request to the AWS metadata service. This token is then used to authenticate our next request to the metadata service for the project variable, which requests the value of the instance's name tag. 

We've also changed the content of our HTML value to use the project variable. The double $$ is necessary to prevent Terraform from interpreting it as a Terraform variable.

Once you apply this, you can see the magic happen:

![tags](/images/tags.png)

---

# Challenge #5

*Learn a bit about Terraform modules, and refactor your vpc.tf to use the [VPC module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest).*

Modules are an important and useful part of Terraform. As you build out a Terraform repo you quickly begin to understand how it can become complex, bloated, and difficult to follow what is pointing where. Modules allow you to abstract functionality to a different layer. Parameters can be passed to the module that will control and alter the behaviour of what it creates, without having to create and define a bunch of resources in your primary repo.

Compare your current `vpc.tf` file with this:

```terraform
data "aws_availability_zones" "available" {}

module "vpc" {
  source             = "terraform-aws-modules/vpc/aws"
  name               = "${var.project}"
  cidr               = "10.0.0.0/16"
  azs                = data.aws_availability_zones.available.names
  public_subnets     = ["10.0.1.0/24"]
  enable_nat_gateway = false
}
```

*Note: You will need to run `terraform init` again in order to install this module before running it.*

This achieves all of the same things. The VPC module will create an internet gateway for your public subnets and place it in the route table. Although don't forget to update your references to your old VPC resource to point to this module instead.

The main differences here are I'm able to more easily configure how many availability zones my VPC uses. I can also create private_subnets which will not have access to the internet gateway in their route table, and in that scenario I could also set `enable_nat_gateway` to true so that they still have a route to the internet.

You'll notice I've also used a data source rather than specify which availability zones to use, this makes the code repeatable across all regions.

You can create your own modules too, and as you gain experience with Terraform you may begin to identify sections of your code that would be better off as a module.