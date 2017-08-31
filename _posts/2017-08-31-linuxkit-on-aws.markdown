---
title:  "Linuxkit on AWS, with Terraform"
date:   2017-08-31 10:00:20 +0000
---

The container revolution is well underway. The Dockeristas swept from their coastal hideouts to infiltrate and disrupt even the most entrenched orders of the old establishment. The democratisation of containers is almost complete and whilst battles rage over cloud orchestration, a new bearded general enters stage-left in the form of Linuxkit. Unassuming and relatively unheralded, Linuxkit could end up providing the bridge between containers and microkernels.

A containerised application is bundled with everything it needs in the way of system libraries and software packages. So why bother replicating those libraries in the host OS? Not only does this mean shifting more bits around, it also leads to an increased surface area for attacks.

[Linuxkit](https://github.com/linuxkit/linuxkit) provides the ability to build an extremely small distribution (~50m) stripped of everything but that which is required to run containers. The init system and all other userland system components (e.g. dhcp, sshd) are containers themselves and as such can be swapped out, or others be plugged in. The whole OS is immutable (unless data volumes are configured), and will run anywhere one finds silicon or a hypervisor: Windows, MacOS, IoT devices, the Cloud etc.

This article hopes to uncover some of the intimacies of Linuxkit, whilst explaining how to build and run one of these minimal distros on AWS using Terraform. The outcome will be an AMI from which we can create an EC2 instance that will run an SSH daemon, and little else.

A Linuxkit machine is represented as a yaml file, and built with the `moby` tool. Here is the yaml configuration for the machine we’ll build (to play along, paste this into a file called `aws.yml`):

```
kernel:
  image: linuxkit/kernel:4.9.39
  cmdline: "console=ttyS0"
init:
  - linuxkit/init:838b772355a8690143b37de1cdd4ac5db725271f
  - linuxkit/runc:d5cbeb95bdafedb82ad2cf11cff1a5da7fcae630
  - linuxkit/containerd:e33e0534d6fca88e1eb86897a1ea410b4a5d722e
  - linuxkit/ca-certificates:67acf038c44bb191ebb704ec7bb39a1524052cdf
onboot:
  - name: sysctl
    image: linuxkit/sysctl:d1a43c7c91e92374766f962dc8534cf9508756b0
  - name: dhcpcd
    image: linuxkit/dhcpcd:17423c1ccced74e3c005fd80486e8177841fe02b
    command: ["/sbin/dhcpcd", "--nobackground", "-f", "/dhcpcd.conf", "-1"]
  - name: metadata
    image: linuxkit/metadata:f5d4299909b159db35f72547e4ae70bd76c42c6c
services:
  - name: rngd
    image: linuxkit/rngd:1516d5d70683a5d925fe475eb1b6164a2f67ac3b
  - name: sshd
    image: linuxkit/sshd:5dc5c3c4470c85f6c89f0e26b9d477ae4ff85a3c
    binds:
     - /var/config/ssh/authorized_keys:/root/.ssh/authorized_keys
trust:
  org:
    - linuxkit
    - library
```

The `moby` tool will snaffle up this yaml and transmute it into a bootable image (such as an ISO).

This yaml describes a set of specialised containers plugged together in a Lego-esque fashion to build a very small OS with just enough nous to run other (normal) containers. What's interesting here is the opportunity for innovation: a savvy developer could elect to write their own init system, add additional system components or even reduce the OS further (maybe they decide they don’t need containerd for instance). 

It is divided into discrete sections, starting with the 'kernel' section that defines which kernel the OS should run. Each kernel is a Docker image containing the kernel along with a tarball of compiled modules. The kernels themselves are based on latest stable releases, with some patches back-ported from newer kernels. That same savvy developer is of course free to compile his or her own customised kernel should they see fit.

The 'init' section defines the Docker images that go together to comprise everything required to get us to a stage where we can run containers. It is followed by the 'onboot' section; a list of images describing short-lived, one-shot services to be run on-boot (such as dhcpd, to get us an IP address) and finally the 'system' section, which will generally be long-running processes that actually give the machine a purpose (for example one could run nginx as a system service). 

So, lets take this yaml definition and turn it into something we can actually run on AWS. To do this we invoke the `moby build` command:

`moby build -name aws -output raw -size 100M aws.yml`

The tool will go off and pull any containers which are not immediately present. It then _unpacks_ the filesystem of each of the containers, does some shifting to make the whole thing palatable to the init process, and then bundles the lot into an initramfs (a compressed cpio archive). The initramfs, along with the kernel and kernel command line, is the build output. It is an entirely immutable Linux machine, with all system services baked in. The default size of 1G is overridden here, and even 100M is probably generous (in all honesty, I couldn't see the relevance of this option once it becomes an AMI).

Let's stop and think about this for a moment: an entirely immutable system, coming in at around 50MB with nothing extraneous to that which is needed to run containers. The root filesystem is read-only, making it stateless and tamper-proof. The build itself takes a matter of seconds, and is eminently reproducible, making it an ideal candidate to pass through a CI system. Although we have added an SSH daemon, by default there is no login (not even a terminal unless you add a getty container). This is starting to feel like the fabled promised-land of _Proper Devops_...

Furthermore it is entirely possible, and probably expected, that you run something like Docker or Kubernetes to enable dynamic scheduling of applications. So we are by no means limited only to the applications which are baked into the image.

In order to run this image on AWS we need to turn it into an AMI. Going back to the `moby build` command above; apart from the name of the resulting image and the yaml defintion, we have also specified a size parameter and an output format. There are several different output formats to choose from, depending on which kind of host system the image is to be run. AWS AMIs require the `raw` output.

We need to take that raw output and convert it to an AMI. Linuxkit makes this easily achievable through the `linuxkit push` command. Under the hood, the `linuxkit push` command does a few things: it uploads the raw disk to an S3 bucket, then initiates an import-snapshot job through VM Import/Export service (http://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-import-snapshot.html) to create an EBS snapshot from that raw disk image. Finally it creates an AMI using that snapshot.

Clearly we’ll need an S3 bucket to upload to, but we also need to configure an IAM role called _vmimport_ and allow the Virtual Machine Import Export service to assume it. 

We’ll use [Terraform](https://www.terraform.io/) to manage these entities as not only is it in the title of this post, but it makes tidying everything up that much easier. So, given a directory structure that now looks like this:

```
.
├── aws.raw
├── aws.yml
└── terraform
    ├── main.tf
    └── files
        ├── assume-role-policy.json
        └── policy.tpl
```

Paste the following into the `assume-role-policy.json` file (this allows the vmie service to assume the vmimport role):

```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
```

And this into `policy.tpl`

```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
            "s3:ListBucket",
            "s3:GetBucketLocation"
         ],
         "Resource": [
            "arn:aws:s3:::${bucket}"
         ]
      },
      {
         "Effect": "Allow",
         "Action": [
            "s3:GetObject"
         ],
         "Resource": [
            "arn:aws:s3:::${bucket}/*"
         ]
      },
      {
         "Effect": "Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource": "*"
      }
   ]
}
```

This is the policy that will be attached to the _vmimport_ role. It allows access to the S3 bucket that Terraform will create, as well as snapshotting and image permissions.

Finally sling this code, which will create the IAM role and S3 bucket, into `main.tf`:

```
provider "aws" {
  region = "us-east-1"
}

data "template_file" "policy" {
  template = "${file("files/policy.tpl")}"
  vars {
    bucket = "${aws_s3_bucket.disk_image_bucket.id}"
  }
}

data "template_file" "containers" {
  template = "${file("files/containers.tpl")}"
  vars {
    bucket = "${aws_s3_bucket.disk_image_bucket.id}"
    key = "aws.raw"
  }
}

################## S3 ###################

resource "aws_s3_bucket" "disk_image_bucket" {
  bucket_prefix = "vmimport"
}

################## IAM ##################

resource "aws_iam_role" "vmimport" {
  name               = "vmimport"
  assume_role_policy = "${file("files/assume-role-policy.json")}"
}


resource "aws_iam_role_policy" "import_disk_image" {
  name   = "import_disk_image"
  role   = "${aws_iam_role.vmimport.name}"
  policy = "${data.template_file.policy.rendered}"
}
```

Running `terraform apply` will result in the creation of an S3 buckets along with the requisite IAM permissions. Note the name of the bucket created, and substitute it in running this command:

`linuxkit -v push aws -bucket <YOUR_BUCKET_NAME> aws.raw`

This may take some time to complete (believe me, it takes even longer if you don't override the default 1G size in the `moby build` command above!). With a bit of luck and a following wind you will eventually be presented with an AMI ID. Use that in the following Terraform code:

```
resource "aws_key_pair" "ssh" {
  key_name   = "ssh"
  public_key = "${file("~/.ssh/id_rsa.pub")}"
}

resource "aws_security_group" "linuxkit" {
  name        = "linuxkit"
}

resource "aws_security_group_rule" "ssh" {
  type              = "ingress"
  security_group_id = "${aws_security_group.linuxkit.id}"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

resource "aws_instance" "linuxkit" {
  count = 1
  ami          = "<YOUR_AMI_ID>"
  instance_type     = "t2.micro"
  key_name          = "${aws_key_pair.ssh.key_name}"
  security_groups             = ["${aws_security_group.linuxkit.name}"]
  associate_public_ip_address = true
}
```

(You can put this in main.tf if you like, although it is now independent of the infrastructure set up before)

This will boot up an AMI in EC2 Classic, which you should be able to log into using your local SSH key (assuming it can be found at ~/.ssh/id_rsa.pub), as root, on its external IP address: `ssh 54.227.66.666 -l root`

Unfortunately you can’t do a great deal, but perhaps that is the point. It is however illustrative to have a poke around, to highlight the properties of a machine built with Linuxkit...

First thing to note is that you are connected to the SSH container, _not_ the machine itself. Files and binaries available to the machine at large are not available within individual containers, unless they are explicitly mounted. 

This can be seen with the aid of a slight detour into an explanation of how Linuxkit handles AWS metadata:

Linuxkit’s metadata package will handle the metadata of VMs booted under different cloud providers. On AWS it extracts a certain amount of information from the [metadata service](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) upon boot, and makes it available under `/var/config` as files. These files can then be bind-mounted into individual containers. An example of this can be seen in our very own `aws.yml`, from above, where the authorized keys are mounted into the root home directory of the ssh container, allowing us to SSH as the root user. 

It’s natural (at least was to me) to try and look in this directory from the command line, however: `ls: /var/config: No such file or directory`.

This is of course because we are in the SSH container. To enter the host OS we need to enter the mount namespace of pid 1 (the init process):

`nsenter --target 1 —mount`

_Now_ we can look in `/var/config`:

```
# ls -1 /var/config
availability_zone
hostname
instance_id
instance_type
local_hostname
local_ipv4
provider
public_ipv4
ssh
userdata
```

It should be noted that the metadata service _is_ still available from within containers (so you can use IAM Instance Profiles, for example). But you’ll have trouble `curl`-ing it, as there’s no `curl` remember? Minimalist OS. (You’d need to add a `curl` container).

Whilst in the parent namespace we can draw notice to another interesting property of a Linuxkit OS: if you run a `df -h` you’ll see that there is _no root filesystem_. However, running the `mount` command yields, amongst other things:

`rootfs on / type tmpfs (ro,relatime)`

Rootfs is a special type of filesystem. On boot, the contents of `initramfs` (which if you recall from above is a cpio archive containing the filesystems of all the containers that make up the OS) is copied into rootfs and mounted on `/`. The kernel then looks for a binary called `init` on this rootfs, which eventually becomes pid 1. On normal Linux systems the _actual_ root filesystem is simply mounted over the top of rootfs and it is forgotten about. But on Linuxkit machines this never happens, the system runs with its root 'disk' in read-only memory, lending the system its immutability. There are no system upgrades (there isn’t even a package manager), if you need to make a change you build a new machine.

Hopefully this has provided a _somewhat_ interesting look at Linuxkit, a project very much in its infancy, which hasn't even yet reached a production milestone. Apart from a few showcase projects (such as a port of Kubernetes) it hasn't attracted an enormous amount of press (or indeed marketing). Also, a minimal container-focussed OS is not a new idea - CoreOS have been doing it for years with Container Linux _and_ can actually boast production workloads. However what I think sets Linuxkit apart is its dogmatic approach to security - truly immutable, read-only operating systems without so much as a terminal - and its potential flexibility; you could see specialised kernels being plugged in for embedded devices, or a customised init system. The system components (e.g. init, containerd) are written in languages which are slightly more accessible than C (e.g. Go or Rust) which again opens up scope for innovation. Furthermore it feels very much like the next logical step in the container revolution: essentially treating the OS like a large container itself. Who knows, if the buzzword magpies decide that microkernels (or unikernels!) are the new shiny thing, Linuxkit stands ready to pounce.
