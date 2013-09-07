---
layout: post
title: "Migrate Redis To A New Server Without Downtime"
date: 2013-09-07 13:25
comments: true
categories: redis
---

There comes a time when you wish to move your redis system from one server to another.  At [CodePen](http://codepen.io) we recently did this to move the redis server to a bigger box.  Below I'll outline the steps we followed to move our redis instances without downtime.

Throughout this walk through, we'll refer to the following terms.

* current master - your currently running redis box.
* candidate master - where you'll be migrating to.
* current slave - your currently running redis slave box.
* candidate slave - the box that will replicate from your candidate master.

This process will take you about 2 hours with setup.  The data migration takes a few minutes.

#Step 1: Set up the two boxes.

We used the [chef-redis](https://github.com/miah/chef-redis) cookbook to build redis from source.  The role with the appropriate attributes overridden follows below.

```ruby
name "redis"
description "redis"

run_list 'recipe[redis::server]'

override_attributes "redis" =>
  {
    'install_type' => 'source',
    'symlink_binaries' => true,
    'init_style' => 'init',
    'config' => {'bind' => '0.0.0.0'}
  }
```

One thing worth noting: you'll need to fork and merge [this](https://github.com/yagince/chef-redis/commit/d7ef9d763ade63f6c62817156e1499fc0d083321) [pull request](https://github.com/miah/chef-redis/pull/52) to allow the `redis-server` and `redis-cli` binaries to symlink properly.  If that pull request has been accepted, you can safely proceed.

#Step Two: Set up replication on the candidate slave.

After having recently set up chained replication on a MySQL server, configuring replication for redis was quite refreshing.  To do so, log onto your candidate slave and issue the following command, remembering to change the 
placeholder below to the real candidate master IP.

```bash
redis-cli
slaveof <candidate_master> 6379
```

Data should not exist on your candidate master, but if for some reason it is, you can verify that the master/slave are synced by issuing the following command.

```bash
redis-cli info | grep '# Replication' -A 10
```

Then look for `master_sync_in_progress:0` in the output.

```
# Replication
role:slave
master_host:ec2-50-***.us-west-2.compute.amazonaws.com
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_priority:100
slave_read_only:1
connected_slaves:0
```

#Step 3: Start the migration.

Now it's time to move your data.  The complexities of this operation are described in a great [article](http://garantiadata.com/blog/real-time-synchronization-tool-for-redis-migration#.Uit9M2SG1Uu) by Garantia Data and are abstracted away with a fantastic [script](https://github.com/GarantiaData/redis-migrate/blob/master/redis-migrate.py) they wrote to make your life great.  The script is short and, being python, very readable.  Understand the concepts and proceed.

To summarize, the script will

* sync master to candidate master, showing progress
* prompt you to turn off the `read only` flag on the the candidate master
* wait for you to re-point your applications to the new candidate master
* prompt you to make the candidate master into a master

The script can move a whole fleet of redis instances if you have them.

So, let's get started. On a server with access to the master and candidate master, get the script

```bash
curl https://raw.github.com/GarantiaData/redis-migrate/master/redis-migrate.py > redis-migrate.py
chmod +x redis-migrate.py
```

and install the python dependencies

```bash
easy_install pip
pip install redis
```

Then run the script

```bash
./redis-migrate --src <current_master> --dst <candidate_master>
```

And follow the on-screen instructions.

After you cut over the applications from the current master to the candidate master, and before you stop synchronization from your master to your slave, be sure to check that no more clients are connecting to the current master.  If you find that you have other connections, be sure you understand what they are doing.  In other words, be sure you have re-pointed all your apps.

```bash
redis-cli client list | awk '{print $1}' | sed s/addr=//g | sed s/:.*//g | sort | uniq
```

And that's it.  We moved about 1.5G of cache in a few seconds because Redis is fast!