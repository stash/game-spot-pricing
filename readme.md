# Complete Satisfactory Server Deployment (CloudFormation)

The template contained within this repository can be used to deploy a Satisfactory server to Amazon Web Services (AWS) in minutes. As the solution leverages "Spot Pricing", the server should cost less than a ten cents an hour to run, and you can even turn it off when you and your friends aren't playing - saving even more money.

## Prerequisites

1. A basic understanding of Amazon Web Services, specifically CloudFormation.
2. An AWS Account.
3. Basic knowledge of Linux administration (no more than what would be required to just use the `wolveix/satisfactory-server` Docker image).

## Overview

The solution builds upon the [wolveix/sattisfactory-server](https://github.com/wolveix/satisfactory-server) Docker image. Support them on Github!

In a nutshell, the CloudFormation template launches an _ephemeral_ instance which joins itself to an Elastic Container Service (ECS) Cluster. Within this ECS Cluster, an ECS Service is configured to run a Docker image. The ephemeral instance does not store any saves, mods, config, data etc. - **all of this state is stored on a network file system: EFS (Elastic File System)**.

The CloudFormation template is configured to launch this ephemeral instance using spot pricing. What is spot pricing you might ask? It's a way to save up to 90% on regular "on demand" pricing in AWS. There are drawbacks however. You're kind-of participating in an auction to get a cheap instance, competing with "on-demand" instances. If demand increases, i.e. someone else puts in a higher bid than you, your instance will terminate in a matter of minutes.

A few notes on the services we're using...

* **EFS** - Elastic File System is used to store config, save games, mods etc. None of this is stored on the server itself, as it may terminate at any time.
* **Auto Scaling** - An Auto Scaling Group is used to maintain a single instance via spot pricing.
* **VPC** - The template deploys a very basic VPC, self-contained and purely for use by the server. This doesn't cost you a cent.
* **Route53** - For rigging up a nice hostname for your server!

## Getting Started

1. Log into AWS, pick a region close to your players.
2. (optional) Set up a Route53 zone if you want
3. Download the `cf.yml` from this repository (later: might host this on S3)
4. Make a new CloudFormation project and upload this `cf.yml`
5. Configure some reasonable values. `t3.xlarge` or the latest variant with 16GiB RAM is ideal. The instance must be x86-64 architecture, not ARM64/Graviton.
6. Set *Server State* to `Running` and wait!

## Next Steps

All things going well, your Satisfactory server should be running in five minutes or so. Wait until CloudFormation reports the stack status as `CREATE_COMPLETE`. Go to the [EC2 dashboard in the AWS console](https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=instanceId) and you should see a Satisfactory server running. Take note of the public IP address. You should be able to fire up Satisfactory, and join via this IP address. No need to provide a port number, we're using Satisfactory's default. *Bonus points* - Public IP addresses are ugly. Refer to Custom Domain Name within Optional Features for a better solution. 

At this point you should *really* configure remote access as per the below section, so that you can access the server and modify files locally if you need.

## Optional Features

### Remote Access

If you know what you're doing, you might want to **SSH** onto the EC2 Linux instance to see what's going on / debug / make improvements. You might also want to do this to upload your existing save. For security, SSH should be locked down to a known IP address (i.e. you), preventing malicious users from trying to break in (or worse - succeeeding). You'll need to create a Key Pair in AWS, find your public IP address, and then provide both of the parameters in the Remote Access (SSH) Configuration (Optional) section.

Note that this assumes some familiarity with SSH. The Linux instance will have a user `ec2-user` which you may connect to via SSH. If you want to upload saves, it's easiest to upload them to `/home/ec2-user` via SCP as the `ec2-user` user (this is `ec2-user`'s home directory), and then `sudo mv` these files to the right location in the Satisfactory installation via SSH.

For remote access, you'll need to:

1. Create a [Key Pair](https://console.aws.amazon.com/ec2/v2/home#KeyPairs:sort=keyName) (Services > EC2 > Key Pairs). You'll need to use this to connect to the instance for additional setup.
2. [Find your public IP address]((https://whatismyipaddress.com/)). You'll need this to connect to the instance for additional setup.

If you're creating a new Satisfactory deployment, provide these parameters when creating the stack. Otherwise, update your existing stack and provide these parameters.

#### Uploading an existing save.

TBD: yolo that file up under `/config/saved` somewhere. Maybe.

### Custom Domain Name

Every time your Satisfactory server starts it'll have a new public IP address. This can be a pain to keep dishing out to your friends. If you're prepared to register a domain name (maybe you've already got one) and create a Route 53 hosted zone, this problem is easily fixed. You'll need to provide both of the parameters under the DNS Configuration (Optional) section. Whenever your instance is launched, a Lambda function fires off and creates / updates the record of your choosing. This way, you can have a custom domain name such as "gameygame.example.com". Note that it may take a few minutes for the new IP to propagate to your friends computers. Have patience. Failing that just go to the EC2 console, and give them the new public IP address of your instance.

## FAQ

**Do I need a VPC, or Subnets, or other networking config in AWS?** 

Nope. The stack creates everything you need.

**What if my server is terminated due to my Spot Request being outbid?** 

Everything is stored on EFS, so don't worry you won't lose anything (well, that's partially true - you might lose up to N minutes of gameplay depending on when the server last saved). There is every chance your instance will come back in a few minutes. If not you can either select a different instance type, increase your spot price, or completely disable spot pricing and revert to on demand pricing. All of these options can be performed by updating your CloudFormation stack parameters.

**My server keeps getting terminated. I don't like Spot Pricing. Take me back to the good old days.** 

That's fine; update your CloudFormation stack and set the SpotPrice parameter to an empty value. Voila, you'll now be using On Demand pricing (and paying significantly more).

**How do I change my instance type?** 

Update your CloudFormation stack. Enter a different instance type.

**How do I change my spot price limit?** 

Update your CloudFormation stack. Enter a different limit. 

**I'm done for the night / week / month / year. How do I turn off my server?** 

Update your CloudFormation stack. Change the server state parameter from "Running" to "Stopped".

**I'm done with Satisfactory, how do I delete this server?** 

Delete the CloudFormation stack.  Except for the EFS, Done.  The EFS is retained when the CloudFormation stack is deleted to preserve your saves, but can then be manually deleted via the AWS console.

**How can I upgrade the Satisfactory version?** 

Every time the server starts, unless you've set "Skip Upgrades", it will download the latest version.

Use the "Use Beta/Experimental" setting (`SteamBeta`) to use pre-releases.

**I can no longer connect to my instance via SSH?** 

Your public IP address has probably changed. [Check your public IP address]((https://whatismyipaddress.com/)) and update the stack, providing your new public IP address.

**I've been hacked!**

That sucks! Fortunately: the ECS task & VPC settings should protect you from major damage, and I've made a best effort to minimize the permissions required, but it's entirely possible I missed something. **You're responsible for reviewing the entire CloudFormation template for shenannigans; it's not my problem and you're doing this at your own risk.**

Some ideas:
* Before you shut down the server, see if any of your infosec / blue-team friends can take a snapshot of the EC2 instance and do forensics. 
  * We could help Coffee Stain / Unreal if it's in their code!
  * Check the `/config` volume for malicious scripts or binaries, since that's the only volume that persists between reboots.
  * It's probably a mod, or serialization. Or weak passwords.
* I'd advise restoring from a known-good save (AWS Backup can help you manage point-in-time backups, for a price). 
  * You could try copy `/config/saved` to a new EFS volume and try to start fresh with a new CloudFormation stack.
* If it's a concern to you, rotate your *PUBLIC* SSH key that you uploaded to AWS; the hacker might have scooped up a copy and could spoof an SSH server.
  * The AWS Systems Manager provides a good alternative to SSH, so you could experiment with using that.

## What's Missing / Not Supported?

* Mods? Waiting on wolveix to add them.
* AWS Backup integration.

## Expected Costs

The two key components that will attract charges are:

* **EC2** - Todo: recalculate on t3.xlarge for 4 hr/day
* **EFS** - Todo: estimate EFS costs, which are hedged by automatic IA tier after 7 days.

AWS does charge for data egress (i.e. data being sent from your server to clients), but again this should (**???**) be barely noticeable.

## Help / Support

1. https://github.com/wolveix/satisfactory-server for the container -- NOTE: other container images not based on this one will probably not work!
2. The upstream https://github.com/m-chandler/factorio-spot-pricing folks
2. You can try open an Issue here, but I will not guarantee you get a response


### Stack gets stuck on CREATE_IN_PROGRESS

It may be because the instance type is not available in your region. As a result of this, an auto-scaling group gets successfully created - however it never launches an instance. This means the ECS service cannot ever start, as it has nowhere to place the container. I would suggest going to the AWS Console > EC2 > Spot Requests > Pricing History, and find a suitable instance type that's cost effective and has little to no fluctuation in price.

In the below example (Paris), `m5.large` looks like a good option. Try to create the CloudFormation stack again, changing the InstanceType CloudFormation parameter to `m5.large`. See: https://github.com/m-chandler/factorio-spot-pricing/issues/10

![Spot pricing history](readme-spot-pricing.jpg?raw=true)

### Restarting the Container

Visit the ECS dashboard in the AWS Console.
1. Clusters
2. Click on the game Cluster
3. Tick the game Service, click Update
4. Tick the "Force new deployment" option
5. Click Next step (three times)
7. Click Update Service

### Basic Docker Debugging

If you SSH onto the server, you can run the following commands for debugging purposes:

* `sudo docker logs $(docker ps -q --filter ancestor=wolveix/satisfactory-server)` - Check the container logs.

DO NOT restart the docker container via SSH. This will cause ECS to lose track of the container, and effectively kill the restarted container and create a new one. Refer to Restarting the Container above for the right method.

## Thanks

Huge thanks to:
* https://github.com/m-chandler/factorio-spot-pricing
* https://github.com/vatertime/minecraft-spot-pricing
* CoffeeStain, ofc
