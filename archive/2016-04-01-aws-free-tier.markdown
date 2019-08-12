---
title:  "Websites on a Shoestring with the AWS Free Tier"
date:   2016-04-01 21:36:20 +0000
---
Mother-in-laws, they’re great aren’t they? My mother-in-law is an enterprising old soul, and runs a couple of businesses. For brownie points, I offered to maintain the websites for these businesses, and some time ago slung them on a VPS. The VPS provider has been surreptitiously hiking their prices ever since, so I thought it’d be a timely exercise to see just how much advantage can be taken of the AWS Free Tier. I’ll also take the opportunity to try out some hipster(-ish) tech, postulate an ill-conceived opinion and foist it upon you, dear reader, as fact.

The websites involved are very low traffic sites, and have no requirement for 100% availability. If they go away for a few minutes, its unlikely anybody would notice. One is a [Rails site][cocojam], the [other PHP][classbase]. I also have a [Wordpress travel blog][ffbr] that I managed to write (_before children_) that is even less regularly viewed, but has sentimental value, so I’ll shift that over too.

3 disparate technologies, co-existing on a single host - sounds like a candidate for containerisation. That, and the fact I can spin up the same services locally, for development, or anywhere else for that matter. The Rails site is already fully Docker-ed up, and the remaining 2 will just share a PHP base container with their source code mounted. I won’t bother covering that process here because blog posts about Docker, well, there are lots.

The AWS Free Tier is the suite of services which can be used before Amazon start charging. It can be used for a whole year before becoming no longer free (I’ll cross that bridge when I come to it, probably move everything to unikernels or whatever is hot at the time). The services which I’ll employ are:

   * EC2
   * RDS
   * ELB
   * ECS
   * ECR
   * VPC

And here is how I’ll hook them up:

![EC2 ECS Diagram](/assets/ec2-ecs-diagram.png)

We’ll use an externally facing ELB to front a single-node ECS cluster. The cluster will run an nginx container to proxy requests to one of 3 backend containers: cocojam, classbase or ffbr. These services themselves are backed by an RDS (mysql) instance. The images for these containers will be stored in ECR, for fast and easy integration.

It should be clear that this is not an HA setup. But, as mentioned, this is not a requirement. The approach taken here could be described as MA (_Mostly_ Available), where we accept that our single node might at any time be terminated (although in practice this does not happen too often). The node consists in an AutoScaling group with a minimum of 1 node and uses an ECS-optimized machine image (AMI). This means that a new box will be automatically launched in the event that we do lose the original, and the combination of the AMI (with all ECS goodies baked in) and images stored in ECR (very close to the box) means the time-to-rebuild is very short. Another advantage of this setup is that it doesn’t preclude itself from becoming HA, all we need to do is dial the AutoScaling group to 2, or more.

ECS is a technology in its relative infancy. It is Amazon’s (belated) answer to the nascent leviathans in the container scheduling arena: Kubernetes, Mesos, CoreOS, even Nomad. Conceptually, tasks are defined as consisting of one or more containers which will be scheduled on the same EC2 host, and services allow you to specify a number of the tasks that should be running at once. The scheduler attempts to run as many instances of the tasks as you specify in the service, and can register containers with an ELB. An ECS agent runs inside a container on each EC2 instance, which is the medium of conversation between the boxes and the ECS scheduler service.

General opinion of ECS is that it has a long way to go to be anywhere near feature parity with the other big players. There are no scheduling policies that can be chosen (even something as simple as ‘run me one task per EC2 instance') although apparently you can write your own. The ELB integration is somewhat clunky: you can only register a single port per EC2 instance, which means you have to be careful around which ports are used by which tasks/containers and you can’t run multiple instances of the same task on the same box. It’d be nicer if you could allow Docker to auto-assign a high numbered port, which is then registered with an ELB. Another niggle is that the ECS agent does not tidy up stopped containers, not a deal-breaker but just amplifies the aura of incompleteness.

So why would you choose ECS then? The pre-baked AMI and easy integration with ECR are 2 good reasons, not to mention IAM and the (albeit clunky) ELB integration. Also because this is Amazon, and if you can make do for now, you sort of know it will come good eventually.

Of course, this being the nineties, one doesn’t just configure all this with points and clicks. One represents one’s infrastructure as code, and for that I chose to use Terraform. The main reason being simply to try it out. It seems to be gaining momentum and many adherents, and I’m a fan of Hashicorp products generally.

With Terraform, you represent desired state in the form of `HCL` - hashicorp markup language, allows comments etc - or `JSON` documents and it maintains the _actual_ state as a file in the same location. This means the tool knows exactly what needs to be applied to achieve desired state from the actual, and can be honest up-front about what changes it plans to apply. The syntax is declarative, meaning that ordering is not important and all files ending .tf in the working directory are included. This keeps things simple, but makes organising code somewhat difficult (there is no ‘include dir/file.tf’ equivalent, and creating modules feels like overkill).

As an illustrative example, here is how I built the ECS cluster. Happily, the Terraform AWS provider does provide an `aws_ecs_cluster` resource. It goes, nice and simply, a little something like this:

```
resource "aws_ecs_cluster" "default" {
  name = “default"
}
```

This will create a Terraform resource which can be referenced throughout the project as `aws_ecs_cluster.default` and will cause an ECS cluster named ‘default’ to be created upon running `terraform apply`. There would be no instances in the cluster, of course, which would render it pretty useless. We can add some:

```
# Create a launch configuration
resource "aws_launch_configuration" "ecs" {
  name = "ecs"
  image_id = "${var.image_id}"
  instance_type = "${var.instance_type}"
  key_name = "${aws_key_pair.default.key_name}"
  iam_instance_profile = "${aws_iam_instance_profile.ecs_instance.id}"
  security_groups = ["${aws_security_group.default.id}"]
  user_data = "#!/bin/bash\necho ECS_CLUSTER=default > /etc/ecs/ecs.config"
}

# ...and apply to an autoscaling group
resource "aws_autoscaling_group" "ecs" {
  name                 = "ecs-asg"
  vpc_zone_identifier  = [ "${module.vpc.subnet_id}" ]
  launch_configuration = "${aws_launch_configuration.ecs.name}"
  min_size             = 1
  max_size             = 1
  desired_capacity     = 1
  load_balancers       = [ "${aws_elb.web.id}" ]
}
```

Without going into too much detail, these 2 stanzas would get me an autoscaling group replete with ECS-tailored launch configuration. The somewhat clumsy variable interpolations (`${thing.other_thing}`) reference attributes of other resources or variables in the same project or in modules included within it. Notice the succinct user data script, all it needs to do is update the /etc/ecs/ecs.config file with the name of the cluster to be joined. This works thanks to the previously described ECS optimised AMI.

Terraform has that evident Hashicorp charm: a single binary, well-documented. properly thought out etc. It ships with a bevy of providers, covering off the usual cast of IaaS outfits: AWS, GCE, Digital Ocean etc. Writing custom providers is made simple, thanks to a well constructed [framework][terraform-providers], and it is of course open-source, meaning a range of additional providers are readily available.

But in some ways this is detrimental. Naturally, it will get compared to Cloudformation; a tool with no shortage of critics, with a syntax so awful that people have built numerous DSLs and abstractions just to spit out the tomes of JSON required to build anything at all. What Cloudformation does have going for it, however, is a team of talented engineers dedicated solely to building a tool for manipulating AWS (a single Iaas), and several years of development. It has built in (often infuriating) safety checks, documented caveats, stack policies and (often infuriating) stack rollback procedures.

Terraform has none of these safety nets. It wins out in a number of areas: speed, instant feedback, far cleaner syntax, available source code, a dry run mode (though this just in - [CF change sets][cf-change-sets]); and is perfect for a side-project such as this. But, even in building such a relatively small subset of infrastructure, I ran into some sticky issues which I ended up resolving by blowing away various things (in one case all the things), a luxury which I’d be unlikely to have in Production, at work. So ultimately I’ll probably be sticking with the devil I know, and keep Terraform tucked conveniently in the back pocket.

In summary then, it was indeed possible to build a rudimentary hosting platform in AWS using solely the Free Tier. But only if you’re willing to accept a smidgeon of downtime when the t2.micro instance is pulled (which it will be, at some point). To minimise the downtime, one solution is to place the instance in an autoscaling group of 1, so that it is immediately replaced. The replacement should be as speedy as possible, and one way to achieve this is to pull down container images from ECR, and run them using the ECS-optimized AMI.

[terraform-providers]: https://www.hashicorp.com/blog/terraform-custom-providers.html  
[cf-change-sets]: https://aws.amazon.com/blogs/aws/new-change-sets-for-aws-cloudformation/
[cocojam]: http://cocojam.co.uk
[classbase]: http://classbase.co.uk
[ffbr]: http://flipflopsandbellyrot.co.uk
