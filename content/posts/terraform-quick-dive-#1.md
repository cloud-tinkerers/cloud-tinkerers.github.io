---
title: "Terraform: Quick Dive #1"
date: 2024-01-14T07:07:07+01:00
language: en
featured_image: ../assets/images/global/cloudtinkerers.png
summary: Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed cursus, odio nec venenatis lacinia, lacus lectus varius nisi, in tristique mi purus ut libero.
description: Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed cursus, odio nec venenatis lacinia, lacus lectus varius nisi, in tristique mi purus ut libero. Vestibulum vel convallis felis. Ut finibus lorem vestibulum lobortis rhoncus.
author: Conor
#authorimage: ../assets/images/global/author.webp
categories: cloud, devops
tags: cloud, aws, infrastructure, infrastructure as code
---
There are many Terraform tutorials out there, not least of which are the official Terraform docs which are actually very good. Most tutorials, however, insist upon you a wealth of knowledge before you touch the tool. This isn't how I like to learn, and maybe it's not how you like to learn, either.

This tutorial aims to get you hands on and tinkering with Terraform in as short a time as possible. Unfortunately there are still a handful of prerequisites before you can actually do the thing. It is important to understand the tools that we use, but I like to build up this knowledge as I go.

**IMPORTANT**: This tutorial may have cost implications. I am opting for AWS free-tier options, but if your account is no longer in the free-tier then it is important to understand that this is a possibility.

# Prerequisites

* An AWS account
* Access keys for your user on your AWS account (best practice is not to use the root account)
* Install the aws-cli - configured with your access keys
* Terraform installed on your PC
* Git installed on your PC
* Some awareness of how to use the CLI

# Setting up

I would encourage you follow this tutorial along yourself, but you can find the end result of the code here: https://github.com/cloud-tinkerers/terraform-projects/tree/terraform-quick-dive-1

Start by creating a new directory for this Terraform project, then create a new file called providers.tf inside of it. Inside the file you'll need to add this:

```terraform
provider "aws" {
  region = "eu-west-1"
}
```

This provider block tells Terraform that we need the aws provider, which is maintained by Hashicorp themselves. I am using the eu-west-1 region, but you can change this to another appropriate AWS region if you desire.

Once you've done this and saved the file, run terraform init inside this directory. You'll see your command line spring to life, and a file called .terraform.lock.hcl will appear inside the directory. You're now ready to begin actually building things.

## Creating a network

Terraform can be a challenging tool as it requires not only knowledge of how to use it, but also knowledge of what you're using it against, in this case AWS. It's common for me to have both the Terraform registry and AWS documentation open for reference when I'm writing Terraform. 

If you've ever built anything in AWS then you know that everything needs to inside a VPC (virtual private cloud). The VPC is our isolated network, within which we can deploy further subnets and infrastructure that relates and talks to each other in order to build an end-to-end application. With that being said, let's create our VPC.

Refer to the [VPC resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc) in the Terraform registry. You can see that this resource has a number of available "arguments" to use, which can alter the behaviour of the resource we're creating. We are just going to use the provided example. Create a new file called vpc.tf and add the following to it.

```terraform
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

This resource block is defining an `aws_vpc` resource which will be known to Terraform as `main`. You can name this differently if you like, but be aware that changing the name will mean you have to use that name every time we reference this resource going forward. In the grand scheme of things the names don't matter in terms of the infrastructure, but a good naming convention can avoid confusion later down the line as your configuration becomes more complex.

Inside our block we've also defined the CIDR block for the VPC, this is the range of private IP addresses that can further carve up and utilise for our infrastructure needs. If you're not sure what is meant by 10.0.0.0/16 then you will want to invest some time into some networking basics at some point (stay tuned for that later).

## Terraform plan

`terraform plan` is the next CLI command we're going to use. This is a useful command to get a picture of what changes Terraform is going to make once the current configuration is applied. Try running this now. You should get output like this:

```terraform
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + arn                                  = (known after apply)
      + cidr_block                           = "10.0.0.0/16"
      + default_network_acl_id               = (known after apply)
      + default_route_table_id               = (known after apply)
      + default_security_group_id            = (known after apply)
      + dhcp_options_id                      = (known after apply)
      + enable_dns_hostnames                 = (known after apply)
      + enable_dns_support                   = true
      + enable_network_address_usage_metrics = (known after apply)
      + id                                   = (known after apply)
      + instance_tenancy                     = "default"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
      + ipv6_cidr_block_network_border_group = (known after apply)
      + main_route_table_id                  = (known after apply)
      + owner_id                             = (known after apply)
      + tags_all                             = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

But there's no point applying this just yet. Let's add a few more things to flesh out our VPC. Add the following to your `vpc.tf` and try to think about what it might do.

```terraform
resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "main"
  }
}
```

A subnet creates a smaller network by taking a piece of the VPC for itself. Our VPC is a /16 which has 65,534 potential IP addresses, whereas this subnet is a /24 which has 254 (technically it would have less than this due to the way AWS works, but let's ignore this for now). Quite a difference, right?

Inside this block you'll note that there's a reference to another resource, our VPC. As mentioned before, if you changed the Terraform name of your VPC then you will need to update the `vpc_id` argument to reflect that. So instead of `aws_vpc.main.id`, it may be something else like `aws_vpc.my_vpc.id`.

If you refer back to the registry page for the VPC resource, you'll see that below the arguments it also has a number of "attribute references". When you create a resource in Terraform these attribute references are available as a way to pass data to other resources or outputs so that you can effectively create tightly coupled resources. Our subnet needs to be part of our VPC, hence why we need to point it at the ID for our VPC. Technically you *could* create the VPC, then use the actual ID to populate this argument, but this is inefficient when Terraform offers you a direct way to chain resources together like this. 

Next we're going to define an "internet gateway", without this, any servers we create inside our subnet will have no route to the internet. This is great for security, but ultimately a bit useless. If the server has no internet access whatsoever then it can only fulfil some fairly niche use cases.

```terraform
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main"
  }
}
```

We're also going to create a route table to attach the internet gateway to:

```terraform
resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "main"
  }
}
```

And we also need to create a route table association to associate our subnet with this route table, and therefore give it access to the internet gateway:

```terraform
resource "aws_route_table_association" "subnet_association" {
  subnet_id      = aws_subnet.main.id
  route_table_id = aws_route_table.rt.id
}
```

You can run another `terraform plan` now, you should see `Plan: 5 to add, 0 to change, 0 to destroy`. - This is the number of resources we have created so far. You'll notice that a lot of the references state (known after apply), this is because Terraform is unable to tell you what these will be until AWS has created the resources.

## Creating infrastructure

Now that we have the bare bones of a VPC, we can put an actual server in it.

Take a look at the [aws_instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance) resource. It has a daunting number of arguments and configuration options, but don't be intimidated. Over time as your experience grows and your uses cases become more complex you'll begin to see when these options are necessary. For the time being it isn't necessary for you to understand everything here.

Create a file called `ec2.tf` and add the following:

```terraform
data "aws_ami" "al2023_latest" {
  most_recent = true
  name_regex = "^al2023-ami-[0-9]{4}.[0-9].[0-9]{8}.[0-9]-kernel-[0-9]+.[0-9]+-x86_64"
  filter {
    name = "architecture"
    values = ["x86_64"]
  }
  owners = ["amazon"]
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.al2023_latest.id
  instance_type = "t3.micro"

  tags = {
    Name = "main"
  }
}
```

In this code we have our first example of a Terraform data source. A data source doesn't create a resource, but it does allow your Terraform resources to read information that is not defined from within Terraform. In this example we are creating a data source for an AMI (Amazon machine image) so that when we create our EC2 resource, we can directly pass it an image to launch from. If you know about AWS then you know that AMI IDs are region specific, but since the filters here only specify a handful of items, this data source will work in any region, even though the output of yours may differ from mine.

This particular data source is searching for the most recent Amazon Linux 2023 AMI that is built for the x86_64 architecture.

**Warning**: This is the first resource we've encountered that may cost money. If you are not in the free-tier then in eu-west-1, a t3.micro instance will cost you $0.0104 hourly (taken from [this useful site](https://instances.vantage.sh/?region=eu-west-1&selected=t3.micro)). You will only be charged per minute that the instance is online for.

Before we apply, we need to configure our EC2 instance a little bit more in order to place it inside our VPC, make it browseable and also to do something at boot time.

```terraform
resource "aws_instance" "web" {
  ami                         = data.aws_ami.al2023_latest.id
  instance_type               = "t3.micro"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.main.id
  vpc_security_group_ids      = [ aws_security_group.main.id ]
  user_data = <<EOF
#!/bin/bash

sudo yum upgrade -y
sudo yum install -y httpd
echo "Hello world!" > /var/www/html/index.html
sudo systemctl start httpd
EOF

  tags = {
    Name = "main"
  }
}
```

*Note*: It is not syntatically necessary to line up your arguments like this...but it does look better.

You'll notice that we've added a number of lines to our web instance resource. We've told it to use the subnet we've defined, to associate a publicly available IP to the instance (there's a small cost to this as well), and also a security group (which we'll get to). But the biggest difference is the user_data. User data is how you pass a boot script to an AWS instance. We've used the "[heredoc](https://developer.hashicorp.com/terraform/language/expressions/strings#heredoc-strings)" format to define EOF as the delimiter, indicating the end of the script.

The script itself is relatively simple. We're updating the yum cache, installed the httpd web server, and creating a very simply HTML page for our server to display. Only, it won't...because we haven't allowed any incoming traffic to this instance yet.

Create a file called `sg.tf` and add the following:

```terraform
resource "aws_security_group" "main" {
  name        = "main"
  description = "Allow http inbound traffic and all outbound traffic"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "main"
  }
}

resource "aws_vpc_security_group_ingress_rule" "allow_http" {
  security_group_id = aws_security_group.main.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 80
  ip_protocol       = "tcp"
  to_port           = 80
}

resource "aws_vpc_security_group_egress_rule" "allow_outbound" {
  security_group_id = aws_security_group.main.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1" # semantically equivalent to all ports
}
```

We've created a security group inside our VPC with 2 rules. One is an ingress rule for port 80, the http port. The other is a rule to allow out outbound traffic from our instance, allowing it to reach the internet.

Finally, we're ready to apply this.

## Terraform apply

I'm sure you've guessed what to run by now, but before you do it, let's add an output to our code so that we know the public IP address that is assigned to our instance.

```terraform
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

It doesn't matter where you put it, but if you want a sensible place I'd put it in the `ec2.tf` file or in its own `outputs.tf` file. Outputs use the same paths as the attribute references we've been using when referring to one resource in another resource. If you ever want to output something specific you can consult the Terraform registry for that resource and see what it has available.

Now you can run `terraform apply`. You should see: `Plan: 9 to add, 0 to change, 0 to destroy`.

Once it has finished applying you should get a message indicating success, as well an IP address. Give your server a few minutes to initalise (you can check on this in the EC2 console if you want) then try curling/browsing to the IP address.

Congratulations! Your website is now available to the world.

To avoid any surprise costs, once you're done admiring your handiwork you can run a `terraform destroy` to get rid of all the resources we've created. This is the beauty of testing in Terraform, you can easily tear it all down once you're finished. But the code is still there should you wish to resume working on it later.

# Next steps

This has barely scratched the surface of what a tool like Terraform can do for you. If you want some direction for what to try next, here are some challenges:

Figure out how to get access to your server via SSH or AWS Session Manager.

Define a variable in your code, and use that variable to name your server.

Investigate the `terraform.tfstate` file and learn more about what Terraform state is.

Add some tags to your instance and figure out how to get the EC2 instance to read this metadata and display it in your HTML page.

Learn a bit about Terraform modules, and refactor your `vpc.tf` to use the [VPC module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest).

If you get stuck, feel free to join the Discord and ask for help.