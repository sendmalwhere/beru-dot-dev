---
title: 'Giving an AWS Packer Builder a Fixed Static IP'
date: 2021-05-20 00:00:00
---

I recently had a really tricky Packer problem, and I don’t think I’m the only one who has had it (or will have it). I wasn't able to find any public references to this problem, so I’d like to explain how I solved it.

## A Packer Predicament
At the company I currently work at, we’re trying to Packerize our server bootstrapping process so that we can produce an AWS AMI as a kind of "baseline" to Terraform new servers from.

We have some servers in data centers, some OpenStack VMs, and some AWS EC2 servers, but common to all of them is an initial setup process that involves running an Ansible job against the server as soon as we bring it up, having it pull down some packages from a private repository, and then doing a bunch of installation and configuration of custom software. This lends itself really well to Packer! The process of building an AMI should just be, theoretically:

1. Stand up a Packer Builder instance
2. Do whatever you need to do to the instance to be able to run the Ansible bootstrapping against it
3. Run the Ansible bootstrap job
4. Zip up the AMI and send it off to whatever AWS accounts need it.

Easy, right?

## Not Easy
The main problem we ran into is specific to our deployment; our private package repos all have firewalls that allowlist the public IP of a new server as part of the pre-Ansible bringup process. But this is a huge problem for us, because the Packer AWS builder is designed to yank a new ephemeral public IP from AWS every time it spins up an instance; there’s no obvious way to maintain a single IP (even if you go so far as to reserve an Elastic IP, there’s no builder config to reach out for it).

**tl;dr** If you have a server that’s necessary for bringup and relies on a non-dynamic IP allowlist, you have a problem that Packer does not have an obvious solution for.

## Trial and Error
The first idea I had for this was to set up a VPC (dedicated to Packer builder instances) with a NAT Gateway, under the assumption that we’d just be able to use the Gateway IP (a static Elastic IP) for the instance to access resources outside the VPC, including our servers. This would just be a single, static source EIP for whatever instance we stick in the VPC. Sounds great, right? And it might be, for some applications—-but for us, this was a nonstarter, because Ansible needs to communicate with the instance, and the NAT Gateway is functionally unidirectional; you could have 4090 instances behind it, so SSH to that EIP is just going to bounce off without a Static NAT back to whatever instance we have, and I felt it would be a waste of effort to try and setup (and tear down) one of those dynamically as part of a Packer Build.

The obvious alternative was to configure a "floating" EIP--a static EIP that would simply get attached to an instance at Packer run-time. So I started to play around with ways to accomplish this.

The configuration looked (essentially) like this before I started messing around with the floating EIP idea:
```hcl
source "amazon-ebs" "aws_builder" {
  access_key        = "${var.aws_access_key}"
  ami_description   = "Base Image - Ubuntu"
  ami_name          = "base_ubuntu_${local.timestamp}"
  instance_type     = "t2.large"
  region            = "us-east-1"
  availability_zone = "us-east-1b"
  secret_key        = "${var.aws_secret_key}"
  source_ami        = "${data.amazon-ami.plain_ubuntu_image.id}"
  ssh_username      = "ubuntu"
} 

build {
  sources = ["source.amazon-ebs.aws_builder"]

  # Some initial setup needed to enable Ansible to run properly
  #
  provisioner "shell" {
    script = "setup_users.sh"
  }

  # Run ansible bootstrapping playbook
  #
  provisioner "ansible" {
    # yada yada yada
    # ....
  }
}
```

My first instinct was to run a `local-shell` provisioner, and simply run a command like this before we do anything else:
```hcl
provisioner "shell-local" {
  environment_vars = ["AWS_ACCESS_KEY_ID=${var.aws_access_key}", "AWS_SECRET_ACCESS_KEY=${var.aws_secret_key}", "AWS_DEFAULT_REGION=us-east-1"]
  inline           = ["aws ec2 associate-address --instance-id ${build.ID} --allocation-id ${var.eip_id}"]
}
```

And this does, in fact, change the IP of the builder instance to the floating EIP! (That is, if you grant your Packer user the `ec2:AssociateAddress` permission). The problem with this is that, if you change the IP of the instance, Packer doesn’t know about the new IP. It’ll keep trying to reconnect to the **original** ephemeral public IP that it grabbed when it first created the instance. So your job will timeout shortly after you run this command. So this `local-shell` script is not a viable solution--at least not by itself.

I then sought to change the configured ssh_host (the parameter for the AWS builder instance’s IP address) _in-flight_ to match the EIP. Basically, I thought, I would set up the instance initially using its ephemeral IP, then assign a new one from my local shell, and finally tell Packer that the instance was at a **new** `ssh_host address`, so that it could carry on starting an Ansible job to the new address.

```hcl
source "amazon-ebs" "aws_builder" {
  access_key        = "${var.aws_access_key}"
  ami_description   = "Base Image - Ubuntu"
  ami_name          = "base_ubuntu_${local.timestamp}"
  instance_type     = "t2.large"
  region            = "us-east-1"
  availability_zone = "us-east-1b"
  secret_key        = "${var.aws_secret_key}"
  source_ami        = "${data.amazon-ami.plain_ubuntu_image.id}"
  ssh_username      = "ubuntu"
} 

build {
  sources = ["source.amazon-ebs.aws_builder"]

  provisioner "shell-local" {
    environment_vars = ["AWS_ACCESS_KEY_ID=${var.aws_access_key}", "AWS_SECRET_ACCESS_KEY=${var.aws_secret_key}", "AWS_DEFAULT_REGION=us-east-1"]
    inline           = ["aws ec2 associate-address --instance-id ${build.ID} --allocation-id ${var.eip_id}"]
  }

  source "source.amazon-ebs.aws_builder" {
    ssh_host = "${var.eip_id}"
  }

  # Some initial setup needed to enable Ansible to run properly
  #
  provisioner "shell" {
    script = "setup_users.sh"
  }

  # Run ansible bootstrapping playbook
  #
  provisioner "ansible" {
    # yada yada yada
    # ....
  }
}
```

But this really, _really_ didn’t have the intended effect. Packer took the additional `ssh_host` variable in the `build` block to indicate that it should spin up its usual instance (with an ephemeral IP), as well as a _parallel_ instance, with the floating EIP assigned to it! This is where HashiCorp’s characteristically sparse documentation really hurt me; they mention ([in this doc](https://www.packer.io/docs/templates/hcl_templates/blocks/source); c.f. "The build-level source block”) that, within a `build` block, you can reference a `source` to assign parameters that are not explicitly defined in the initial configuration. This _really sounds like_ it would allow you to dynamically alter parameters, but that seems to be pretty inconsistent across builders.

It was starting to feel like even the notion of attaching a floating EIP was hopeless with the tooling provided out of the box by Packer and the AWS EBS builder source. At wit’s end, I finally decided to go down a path suggested by [an old Google Groups post](https://groups.google.com/g/packer-tool/c/G6FEjknALSs?pli=1). It felt like a hack, but it was also very clear to me that there was no other way.

## `cloud-init` Saves The Day
The basic structure of the solution looks like this:

- Reserve a single EIP and allowlist it for the remote sources that aren’t so friendly to dynamic auth;
- Give the Packer user’s AWS IAM policy some enhanced permissions so that it can associate an EIP to an EC2;
  - You’ll also want to create an Instance Profile based off of this policy, and then ensure that your Packer Builder Instance launches with this profile attached to it;
- Create a user_data file with cloud-init config and make sure your Packer Builder Instance launches with that user_data file;
- Configure the Builder Instance to launch with `ssh_host = "${var.your_eip_address}"`.

I’m going to walk you through what this config looks like from the AWS side, as well as from the cloud-init config side.

### Permissions, Packer, and Other Perfidy
Let’s talk about those "enhanced permissions" for the Packer user’s IAM policy first.

Packer helpfully provides [the JSON for an IAM role sufficient for minimal Packer permissions](https://www.packer.io/docs/builders/amazon#iam-task-or-instance-role). This is sufficient for regular Packer builds with an EBS builder, but we actually need to add a few extra permissions to make this floating EIP scheme work.

You’ll need to give the Packer user’s policy three extra permissions:

- `ec2:AssociateAddress` (Write-level access)
- `iam:GetInstanceProfile`
- `iam:PassRole`

The first of these is literally to run the command associating the EIP to the instance, and the latter two are related to Instance Profile permissions (`GetInstanceProfile` allows your Packer user account to access the instance profile you’ll be creating, and `PassRole` is an idiosyncratically granular AWS requirement for actually using an instance with that role).

### Cool; what about the Instance Profile, now?
Your user-script might be configuring other stuff (ours sure is). But, to associate a new IP address to the instance running the command, your user-script merely needs the following config (with `eipalloc-#############` replaced by the actual ID of your EIP allocation):

```
#cloud-config
packages:
  - awscli

runcmd:
  - AWS_DEFAULT_REGION=us-east-2 aws ec2 associate-address --instance-id $(curl -s "http://169.254.169.254/latest/meta-data/instance-id") --allocation-id eipalloc-#############
```

If all of the above configuration looks trivial, know that it took me almost an hour to figure out all of the nonsense related to YAML, self-reference of AWS metadata, and permissions.

## tl;dr: A Working Builder Definition
Here’s the AWS EBS builder definition that actually does the thing (as HCL2; JSON config in HashiCorp products is for gibbering heathen poltroons):

```hcl
source "amazon-ebs" "aws_builder" {
  access_key        = "${var.aws_access_key}"
  ami_description   = "Base Image - Ubuntu"
  ami_name          = "base_ubuntu_${local.timestamp}"
  instance_type     = "t2.large"
  region            = "us-east-1"
  availability_zone = "us-east-1b"
  secret_key        = "${var.aws_secret_key}"
  source_ami        = "${data.amazon-ami.plain_ubuntu_image.id}"
  ssh_username      = "ubuntu"
  user_data_file    = "user_data" # The filename (and/or path to the filename) of the user-script configured above
  ssh_host          = "${var.eip_address}" # The floating EIP, i.e., the static persistent IP address
  iam_instance_profile = "Packer_Instance_Profile" # The *name* (not ID) of the Instance Profile configured above
}
```

Thanks for reading--I hope this helps someone out there!
