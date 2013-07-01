---
layout: post
title: "Painless AWS Autoscaling With EBS Snapshots And Capistrano Part 2"
date: 2013-06-29 05:12
comments: true
categories: aws
---


##Part 2

This is part two of a series designed to get your auto scaling environment running.  If you're just tuning in, check out [part 1](/blog/2013/06/28/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano)

##Catching Up

In the last part of this series, we did a bunch of manual key mashing to take our first snapshot.  This gives us the foundation we need to automate the the process.  In this part we will review the scripts required to make auto scaling work as expected.  Also, at the end of this post, I'll share the Chef recipe used to install all the scripts described here.

##The Scripts

1. `snapshot.py` - a python script to snapshot a volume on deploy
1. `deploy:snapshot` - a capistrano task used to call `snapshot.py`
1. `prep_instance.py` - a python script to mount a volume from the most recent snapshot and tag the instance.
1. `utils.rb` - a ruby script used during `cap deploy` to get instance dns names by tags.
1. `production.rb` - a the file used by capistrano multistage to get a list of servers.
1. `chef_userdata.sh` - a userdata script for bootstrapping chef.
1. `userdata.sh` - a userdata script that does not include chef bootstrapping.
1. `autoscaling` - a chef recipe used to set up all the scripts above.

##Before you start

Let's review the tool set we'll be working with.  So far, we've used bash and the [AWS Command Line Tools](http://alestic.com/2012/09/aws-command-line-tools) and we've done just fine.  We'll still use bash to stitch together our scripts, but in the next few steps we'll be using both python and ruby to accomplish our goals.  I find python to be more expressive and capable than bash when dealing with lots of variables that need to be type checked and have default values.  And ruby is a good fit for Capistrano.  So, we'll be using the [boto](https://pypi.python.org/pypi/boto) libraries on the server side, and the [AWS SDK for Ruby](http://aws.amazon.com/sdkforruby/) on the client (Capistrano) side.

I use Chef to manage my dependencies, but if you're doing this by hand, the AMI on which these scripts will run must have the boto libraries pre-installed.  To do so, you can issue the following statements.

```bash
sudo apt-get install python-setuptools
sudo easy_install pip
sudo pip install boto
```

Also, boto expects on a `.boto` file to exist in the home directory for the user who executes these scripts.  We'll set the `BOTO_HOME` variable in our driver script later on in this post.

The rest of this part will describe the file you need and what they do.

##snapshot.py

[source](https://gist.github.com/tsabat/5890733)

The python script we review here will, given an instance ID, look up the volume attached to a device and take a snapshot of it.  It also tags the snapshot, so that future scripts can query the tags.  This script is called at deploy time so that the most recent code is always ready to mount on an auto scaling instance.

The `parsed_args` method at the top of the script does a decent job of describing its default values.  You'll probably want to change the `--tag` argument to match your organization's needs.

In the `main` method we do all our work.  The line

```python
vols = conn.get_all_volumes(filters={'attachment.instance-id': args.instance_id})
```

drives this little app.  We search for instance IDs that match that of the calling box.

Then we iterate over the volumes, searching for the mount point (device) we set up earlier.

Once found, we tell the script to create the snapshot and add the tag.

```python
snap = code_volume.create_snapshot(snapshot_description(code_volume, args.instance_id))
snap.add_tag('Name', args.tag)
```

And that's it.  We'll use this script later on in our automation.

##deploy:snapshot

The Capistrano task below calls `snapshot.py` on deployment.

```ruby
 task :snapshot do
  desc "Take a snapshot of the codepen volume for future autoscaling needs"
  run "BOTO_CONFIG=/home/deploy/.boto /home/deploy/snapshot.py"
 end
```

Notice the presence of the `BOTO_CONFIG` environment variable.  The boto library provides [documentation](https://code.google.com/p/boto/wiki/BotoConfig) for the appropriate keys to add to this INI-style file.

Finally, remember to add the snapshot task to your `after_deploy` hooks in your Capistrano `deploy.rb`.

```ruby
after :deploy, "deploy:snapshot"
```

##prep_instance.py

[source](https://gist.github.com/tsabat/5890808)

The script we'll review here will, given a tag, search for the most recent snapshot, create a volume and mount it.  Furthermore, the script will apply tags to the instance itself.  We'll use these tags in our Capistrano ruby script.

As with the other python script, there is a `parsed_args` method that defines the default values we'll need.  The `help` section of each describes each default.  The pair that need a bit more explaining are `device_key` and `device_value`.  If you recall in Step 4 of [part one](/blog/2013/06/28/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano/) of this series, device names can differ from AWS to your OS.  These two arguments compensate for this fact.

Some interesting parts of the code include `wait_fstab` and `wait_volume`.  Both deal with the fact calls to create volumes, snapshots, and to attach devices are async.  So, we must poll the API, waiting for the status we expect.  For example, in the snippet below, our script sleeps for up to 60 seconds until the status we want appears.  If not, it throws an exception.

```python
def wait_volume(conn, args, volume, expected_status):
    volume_status = 'waiting'
    sleep_seconds = 2
    sleep_intervals = 30
    for counter in range(sleep_intervals):
        print 'elapsed: %s. status: %s.' % (sleep_seconds * counter, volume_status)
        conn = ec2.connect_to_region(args.region)
        volume_status = conn.get_all_volumes(volume_ids=[volume.id])[0].status
        if volume_status == expected_status:
            break
        time.sleep(sleep_seconds)
 
    if volume_status != expected_status:
        raise Exception('Unable to get %s status for volume %s' % (expected_status, volume.id))
 
    print 'volume now in %s state' % expected_status
```

##utils.rb

[source](https://gist.github.com/tsabat/5890996)

This tool grabs all instance DNS names from aws.  We use this in the Capistrano multistage `production.rb` to get an array of dns names.  It is pretty self-explanatory.  Since this script will be distributed to your developers, it would probably be a good idea to lock the credentials down to read-only.  You will have to require this in your `deploy.rb` like so

```ruby
require './config/deploy/utils'
```

Here's the file itself.  This makes deployment nice because it dynamically grabs EC2 Instances tagged with the Role and Environment you specify along with an `instance-state-name` of running.  This guarantees that you're pushing out to all the servers.

```ruby
require 'aws-sdk'
require 'awesome_print'
 
class AwsUtil
 
  def ec2_object
    # deployer user, has read-only access
    AWS::EC2.new(
      access_key_id: "<your_key_here>",
      secret_access_key: "<your_secret_here>",
      region: 'us-west-2'
    )
  end
 
  def deployed_app_server_dns_names
    ec2_object.instances.
      filter('tag:Role', 'app').
      filter('tag:Environment', 'production').
      filter('instance-state-name', 'running').
      map {|r| r.dns_name}
  end
 
  def print_dns_names
    ap deployed_app_server_dns_names
  end
end
```

##production.rb

[source](https://gist.github.com/tsabat/5891043)

The [capistrano multistage](https://github.com/capistrano/capistrano/wiki/2.x-Multistage-Extension) extension allows you to specify a file for each deployment target.  This script replaces `production.rb` and calls out to `utils.rb` to get dns names.

```ruby
set :branch, "master"
 
# tagged:
# Role:app && Environment:'production'
# filtered:
# instance-state-name:running
aws_servers = AwsUtil.new.deployed_app_server_dns_names
 
role(:app) { aws_servers }
role (:web) { aws_servers }
role :db, aws_servers[0], primary: true
```

##chef_userdata.sh

[source](https://gist.github.com/tsabat/5891084)

This file will be passed to an autoscale launch config.

The shebang line uses the `-ex` args to instruct bash to exit on error and to be very verbose when executing.  This is super-handy for debugging your user data script.

```bash
#!/bin/bash -ex
```

The `exec` call redirects standard out and error to three different places.

```bash
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
```

We slightly shorten the dns name and assign it to the `EC2_HOST` variable.

```bash
EC2_HOSTNAME=`ec2metadata --public-hostname`
EC2_HOST=`echo $EC2_HOSTNAME | cut -d. -f1`
EC2_HOST=$EC2_HOST.`echo $EC2_HOSTNAME | cut -d. -f2`
```

If you're not using chef, you can skip the following bits.  However, if you are using Chef you can boostrap the node this way: delete the `.pem`, set up a `first-boot.json` file, and pass the `EC2_HOST` variable to the `client.rb` file so your chef node name is useful.

This script also assumes that the chef libraries are already installed and had been bootstrapped once before.

```bash
if [ -a /etc/chef/client.pem ]; then
  rm /etc/chef/client.pem
else
  echo "no pem to delete"
fi
echo '{"run_list":["role[app_server]","recipe[passenger]","recipe[autoscaling]"]}' > /etc/chef/first-boot.json
sed -i "s/node_name \".*\"/node_name \"app_$EC2_HOST\"/g" /etc/chef/client.rb
sudo chef-client -j /etc/chef/first-boot.json
```

And finally we call `userdata.sh`

##userdata.sh

[source](https://gist.github.com/tsabat/5891225)

Ultimately, this is the script that does all the work.  It mounts drives as described in Step 4 of [part one](/blog/2013/06/28/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano/) and then calls `prep_instance.py` from above.

Although this script is mighty important, we've covered all the details elsewhere.  Look it over and you'll recognize parts.

##autoscaling chef recipe

[source](https://github.com/tsabat/autoscaling)

We've reviewed a lot of scripts here in this document.  You may be wondering where to put them all.  Chef to the rescue!  Even if you're not using Chef, the [default recipe](https://github.com/tsabat/autoscaling/blob/master/recipes/default.rb) from my recipe creates a great guide for placing these files where you want them.

Here's an example from the `default.rb`.  In this case `/root/.boto` is where we're going to place the [boto.cfg.read_only.erb](https://github.com/tsabat/autoscaling/blob/master/templates/default/boto.cfg.read_only.erb) file.

The `owner`, `group` and `mode` functions should make sense to you.

```ruby
template "/root/.boto" do
  source "boto.cfg.read_only.erb"
  owner "root"
  group "root"
  mode "0660"
end
```

This pattern is repeated throughout the document.

##Fin

You've reached the end of this part.  So far, you've reviewed all the scripts you'll need to auto scale your environment.  In [part 3](/blog/2013/06/29/painless-aws-autoscaling-with-ebs-snapshots-and-capistrano-part-3) we'll look at some bash scripts for setting up your autoscaling rules, and review where all these scripts go.