---
layout: post
title: "Painless AWS Auto Scaling With EBS Snapshots And Capistrano"
date: 2013-06-28 09:10
comments: true
categories: aws 
---

<div style="width: 250px; float: right; margin: 0 0 10px 10px; padding: 20px; border: 1px solid #ccc;">
  <h4>A Three Part Series:</h4>
  <ul>
    <li><a href="http://boomboomboom.biz/blog/2013/06/28/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano/">Part 1</a></li>
    <li><a href="http://boomboomboom.biz/blog/2013/06/29/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano-part-2/">Part 2</a></li>
    <li><a href="http://boomboomboom.biz/blog/2013/06/29/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano-part-3/">Part 3</a></li>
  </ul>
</div>

###Choices to Make

[AWS](http://aws.amazon.com/) (Amazon Web Services) auto scaling is a simple concept on the surface: You get an AMI, set up rules, and the load balancer takes care of the rest. However, actually getting it done is more complicated. 

Some choices are worse than others: you could bake an AMI (Amazon Machine Image) before you deploy, but that could add 10 minutes or more to each deployment. Some are dangerous: you could create an AMI after each deploy, but you run the risk that an auto scale even happens before your AMIs are done. Plus, you have a whole variety of AMIs deployed in at any given time. Some are similar to what we propose in this tutorial: you could push your code to S3 on each deploy and have user-data scripts that pull it down on each auto scaling event.  However you slice it, to get auto scaling to fit into your development work flow in a transparent way takes careful thought and planning.  

We recently rolled out the following solution at [CodePen](http://codepen.io/). It keeps our AMIs static and our application ready for scaling on EBS (Elastic Block Store) snapshots. We can push code using [Capistrano](https://github.com/capistrano/capistrano) and let a few scripts distribute the ever-changing code base to our fleet of servers. I'd like to share the steps required to make it work.  This series of posts will walk you through the steps required to build an auto-scaling infrastructure that stays out of your way.  

###Overview

The process can be summed up like this:

* Source is mounted on an EBS Volume.
* Snapshots are taken on deployment.
* When AWS scales up, new instances are started from latest snapshot.
* Instances are tagged with roles so that deployment scripts always push code to the right servers.


###Before you start

This walk through assumes that you have a working Capistrano deployment going on AWS. If you need some help with that, the guys at [Beanstalk](http://beanstalkapp.com/) have a [great guide](http://guides.beanstalkapp.com/deployments/deploy-with-capistrano.html) for getting started.  We use the Capistrano [Multistage](https://github.com/capistrano/capistrano/wiki/2.x-Multistage-Extension) to separate our our deployment environments.

Also, it is a good idea to practice this whole setup on a clone of your application environment. Hopefully you're running your instance on an EBS-mounted root partition so you can simply create an AMI and run these steps in a safe environment.

A functional AWS API Tools environment is a requirement as well, because this walkthrough will use them extensively. Although I do my development on a Mac, I prefer a Linux environment for this type of work. I keep an EBS-backed micro instance around for all my admin work. I found Eric Hammond's instructions for [installing aws command line tools](http://alestic.com/2012/09/aws-command-line-tools) invaluable for this task.  


###Identifying Your Environments

You'll be working in two environments for this tutorial.

1. Workstation Environment - this is where you have the AWS API Tools installed. A micro instance is nice for this.
1. Instance Environment - this is the instance where you deploy your code.  To follow along with this guide, the Instance Environment should have a working Rails environment in the Source directory.  In this case, that's `/home/deploy/codepen`. Yours will obviously be elsewhere.

I'll reference these two environments throughout this walk through.


##Step 1: Create the EBS volume in AWS - Workstation Environment

In this section, we'll do the EBS legwork to get your code snapshot-ready.

First, let's identify where your source lives.  Capistrano's `deploy.rb` defines your Source directory with the `:deploy_to` setting.  We'll refer to this as your Source from here on out.

```ruby
# Source
set :deploy_to, "/home/deploy/codepen"
```

You will mount your source directory on an EBS volume in a process similar to the instructions laid out in the Amazon article [Running MySQL on Amazon EC2 with EBS](http://aws.amazon.com/articles/1663). This is a manual process for now, but we'll automate this with a script later on in the article.  

Let's create a volume using the command line tools.

```bash
VOL_ID=`ec2-create-volume --size 5 --availability-zone us-west-2c | awk '{print $2}'`
echo $VOL_ID
```

Now let's poll for your volume status. Repeat the command below until your echo returns `available`. It is worth noting that AWS calls are asynchronous. This means that even though you asked AWS to create the volume, you can't use it until its `Status` becomes `available`. That's what we're doing here.

```bash
STATUS=`ec2-describe-volumes $VOL_ID | awk '{print $4}'`
echo $VOL_ID
```


##Step 2: Get the Instance ID - Instance Environment

You also need your Instance ID in order to mount this volume, so let's get that. You will need to be logged into the machine.

```bash
INSTANCE_ID=`ec2metadata --instance-id`
echo $INSTANCE_ID
```

I take for granted here that the `ec2metadata` command is available on Ubuntu Cloud Instances. If you're using some other flavor of OS, you can do:

```bash
INSTANCE_ID=`curl http://169.254.169.254/latest/meta-data/instance-id`
```

Remember your `$INSTANCE_ID`for the next section.


##Step 3: Mount the Volume - Workstation Environment 

In the previous section you got your instance ID. Let's put that in a variable on the workstation so you can easily access it.

```bash
$INSTANCE_ID=<your instance id here>
```

Now, let's ask AWS to mount this volume to your instance on device `/dev/sdf`.

```bash
ec2-attach-volume $VOL_ID -i $INSTANCE_ID -d /dev/sdf
```


##Step 4: Mount the File Systems to the Volume - Instance Environment

In the previous steps, we attached a volume to an instance. Now we're on the instance and we'll associate that volume with the file system.

First, verify that the device exists.

```bash
ls /dev/xvdf
```

It's worth noting that the device I asked AWS to mount, `/dev/sdf`, is not the same as the device we're checking for.  Ubuntu uses the prefix `xvd` instead of `sd` to enumerate devices.  So, we search for `/dev/xvdf` to see that the `ec2-attach-volume` call worked. 

It can take some time for the device to mount.  During that time, the command above could return `No such file or directory`. Just keep trying.

Now create an xfs filesystem on the device.

```bash
sudo apt-get update
sudo apt-get install -y xfsprogs

grep -q xfs /proc/filesystems || sudo modprobe xfs
sudo mkfs.xfs /dev/xvdf
```

In the call above, we asked apt to install the `xfsprogs` package, we test that xfs was installed.  Then we make the filesystem with the `mkfs.xfs` command.

We'll create a temp at `/tmp/mount.sh` that you can [grab from here](https://gist.github.com/tsabat/5887028#file-mount-sh) 

Let's review what it does.  Lines 1 - 6 below echo our mounting instructions into fstab. We want to mount our device `/dev/xvdf` to the file system at `/cp`.  Furthermore we want to mount the directory `/home/deploy/codpen/` to `/cp/codepen/`. The second mount just acts like a symlink, pointing the home directory of the deploy user to the mounted filesystem. The juicy bits are below.

```bash
grep "codepen_fstab_setup" /etc/fstab
if [ $? -eq 1 ]; then
    echo "# codepen_fstab_setup" | tee -a /etc/fstab
    echo "/dev/xvdf /cp xfs noatime 0 0" | tee -a /etc/fstab
    echo "/cp/codepen /home/deploy/codepen     none bind" | tee -a /etc/fstab
fi
```

Then, lines 1 - 12 below make the directories if they don't exist, and finally line 18 calls `mount -a`.  This tells the OS to run the `mount` command against `/etc/fstab`, effectively running the configuration we just set up.

```bash
if [ $? -eq 0 ]; then
    if [ ! -d /cp ]; then
        mkdir -m 000 /cp
    fi
 
    if [ ! -d /home/deploy/codepen ]; then
        mkdir -m 000 -p /home/deploy/codepen
    fi
    mount -a
else
    echo FAIL
fi
```

If you have mounted `/dev/xvdf` and downloaded and executed `mount.sh` then you can verify that your devices and directories are mounted and linked by issuing the `mount` command.

```bash
mount

...snip
/dev/xvdf on /cp type xfs (rw,noatime)
/cp/codepen on /home/deploy/codepen type none (rw,bind)
```

Now you have your source directory hosted on an EBS volume.


##Step 5: Verify, Deploy and Snapshot - Workstation Environment

Now your code is ready for deployment. Let's verify that everything is in place.

```bash
cap deploy:check
```

A hangup here could be permissions. If your code was already deployed to the Source directory, the above steps should have simply linked your code in Source to the `/cp/codepen` directory.  If for some reason this did not happen, you can initialize your deployment now.

```bash
cap deploy:setup
cap deploy
```

With a successful deployment, you're ready to snapshot.

```bash
SNAPSHOT_ID=`ec2-create-snapshot -d "First snapshot" $VOL_ID | awk '{print $2}'`
```

We're also going to tag the snapshot.  This step is important becasue during the launch of a new box, we'll search for the latest snapshot with this tag name and mount it as our Source directory.

```bash
ec2-create-tag $SNAPSHOT_ID --tag Name="codepen-app"
```

##Done, for now.

In [part 2](/blog/2013/06/29/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano-part-2) of this series, we'll automate what we did here with a script.
