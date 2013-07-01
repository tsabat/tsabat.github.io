---
layout: post
title: "Painless AWS Autoscaling With EBS Snapshots And Capistrano part 3"
date: 2013-06-29 09:47
comments: true
categories: 
---

This is part three of a series designed to get your auto scaling environment running.  If you're just tuning in, check out [part 1](/blog/2013/06/28/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano) and [part 2](/blog/2013/06/29/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano-part-2)

In the last part of this series, we reviewed a bunch of scripts used to deal with properly snapshotting and mounting volumes.  In this part we'll get our auto scaling system set up in aws.  Then we'll give a high-level run-through of what you need to do to complete your setup.  In this part we'll review these scripts:

1. `aws_create_lb.sh` - a bash script for creating an load balancer.
1. `aws_create_launch_config.sh` - a bash script for creating launch configs.
1. `aws_create_autoscaling_group.sh` - a bash script for creating an autoscaling group.
1. `aws_create_scaling_policies.sh` - a bash script for creating policies and alarms.

Finally, I'll tie together the whole process, referring to scripts as I go.

###Before you start

The scripts that we're about to execute will work out of the box, but will have some very codepen-specific stuff listed in them.  You'll probably want to do your own naming of policies, lbs, etc.

Also, I'm not going to go into great detail about the AWS creation scripts.  The most useful article I found on the topic is on the [cardinal path](http://www.cardinalpath.com/autoscaling-your-website-with-amazon-web-services-part-2/) blog.  I followed his instructions until I understood the process well enough to build my own.

##AWS Autoscaling Creation Scripts

I wrote bash scripts to automate the creation of my autoscaling setup.  Let's review each in turn below.

[aws_create_lb.sh](https://gist.github.com/tsabat/5891540) - It is pretty obvious what's happening in this script.  Be sure to change the `CERT_ID` variable and the `LB_NAME` to something that makes sense for you.

[aws_create_launch_config.sh](https://gist.github.com/tsabat/5891427) - Here we're building our launch config.  Be aware that the `USER_DATA_FILE` here is the one we created in [part 2](/blog/2013/06/29/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano-part-2) of this walkthrough.  Find the source [here](https://gist.github.com/tsabat/5891084).

[aws_create_autoscaling_group](https://gist.github.com/tsabat/5891536) - Again, boilerplate stuff.

[aws_create_scaling_policies.sh](https://gist.github.com/tsabat/5891535) - Note that I'm creating four things in this script:  two policies and two metric alarms.  To keep things DRY, I wrote a function and overrode the global variables twice, like so:

```bash
echo "scale up policy"
POLICY_NAME="scale-up"
ADJUSTMENT=1
SCALE_UP=$(add_policy)
 
echo "scale down policy"
POLICY_NAME="scale-down"
ADJUSTMENT="-1"
SCALE_DOWN=$(add_policy)
```

In this case, `add_policy` is a function that we call twice.

##High Level Overview

I've thrown a lot of information at you all at once here and it's time to review the setup details from a high level.

* Create a new instance.  Install your webserver and get Capistrano pushing code.
* Bootstrap your node with Chef, if you're using it
* Create an AMI to work with
* Launch a fresh AMI to work through your process
* Manually set up your mount and volume for the first time, and snaphshot it, as described in [part 1](/2013/06/28/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano).
* Put `snapshot.py` and `prep_instance.py` onto your production AMI.
* Add the `deploy:snapshot` task into your `deploy.rb`.
* Put `utils.rb` into your `config` dir in your rails setup.
* Add `production.rb` to your Capistrano multistage setup at `config/environments`
* Keep `chef_userdata.sh` and `userdata.sh` handy for when you work with `aws_create_launch_config.sh`
* Review the `autoscaling` recipe to see where all these files are stored on your production instance.

##Conclusion

Setting up an environment that scales based on load is cumbersome if you must constantly build AMIs every time your code changes.  The instructions listed in this series make Capistrano deployment a natural part of your autocaling process.