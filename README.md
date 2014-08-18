# Multinode Ceph on Vagrant

This workshop walks users through setting up a 3-node [Ceph](http://ceph.com) cluster and mounting a block device, using a CephFS mount, and storing a blob oject.

It follows the following Ceph user guides:

* [Preflight checklist](http://ceph.com/docs/master/start/quick-start-preflight/)
* [Storage cluster quick start](http://ceph.com/docs/master/start/quick-ceph-deploy/)
* [Block device quick start](http://ceph.com/docs/master/start/quick-rbd/)
* [Ceph FS quick start](http://ceph.com/docs/master/start/quick-cephfs/)
* [Install Ceph object gateway](http://ceph.com/docs/master/install/install-ceph-gateway/)
* [Configuring Ceph object gateway](http://ceph.com/docs/master/radosgw/config/)

## Install prerequisites

Install [Vagrant](http://www.vagrantup.com/downloads.html) and a provider such as [VirtualBox](https://www.virtualbox.org/wiki/Downloads).

We'll also need the [vagrant-cachier](https://github.com/fgrehm/vagrant-cachier) and [vagrant-hostmanager](https://github.com/smdahlen/vagrant-hostmanager) plugins:

```console
$ vagrant plugin install vagrant-cachier
$ vagrant plugin install vagrant-hostmanager
```

## Add your Vagrant key to the SSH agent

Since the client machine will need the Vagrant SSH key to log into the server machines, we need it add it to our local SSH agent:

```console
$ ssh-add -K ~/.vagrant.d/insecure_private_key
```

## Start the VMs

This instructs Vagrant to start the VMs and install `ceph-deploy` on the client.

```console
$ vagrant up
```

## Create the cluster

We'll create a simple cluster and make sure it's healthy. Then, we'll expand it.

First, we need to get an interactive shell on the client machine:

```console
$ vagrant ssh ceph-admin
```

The `ceph-deploy` tool will write configuration files and logs to the current directory. So, let's create a directory for the new cluster:

```console
ceph-admin$ mkdir test-cluster && cd test-cluster
```

Let's prepare the machines:

```console
ceph-admin$ ceph-deploy new ceph-server-1 ceph-server-2 ceph-server-3
```

Now, we have to change a default setting. For our initial cluster, we are only going to have two [object storage daemons](http://ceph.com/docs/master/man/8/ceph-osd/). We need to tell Ceph to allow us to achieve an `active + clean` state with just two Ceph OSDs. Add `osd pool default size = 2` to `./ceph.conf`. Now, it should look similar to:

```
[global]
fsid = c4e2f4d3-5c2b-4421-80a2-042527977269
mon_initial_members = ceph-server-1, ceph-server-2, ceph-server-3
mon_host = 172.21.12.11,172.21.12.12,172.21.12.13
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true
osd pool default size = 2
```

## Install Ceph

We're finally ready to install!

```console
ceph-admin$ ceph-deploy install ceph-server-1 ceph-server-2 ceph-server-3
```

## Configure monitor and OSD services

Next, we add a monitor node:

```console
ceph-admin$ ceph-deploy mon create-initial
```

And our two OSDs. For these, we need to log into the server machines directly:

```console
$ vagrant ssh ceph-server-2 -c 'sudo mkdir /var/local/osd0'
```

```console
$ vagrant ssh ceph-server-3 -c 'sudo mkdir /var/local/osd1'
```

Now, back on our admin machine, we can prepare and activate the OSDs:

```console
ceph-admin$ ceph-deploy osd prepare ceph-server-2:/var/local/osd0 ceph-server-3:/var/local/osd1
ceph-admin$ ceph-deploy osd activate ceph-server-2:/var/local/osd0 ceph-server-3:/var/local/osd1
```

## Configuration and status

We can copy our config file and admin key to all the nodes, so each one can use the `ceph` CLI.

```console
ceph-admin$ ceph-deploy admin ceph-admin ceph-server-1 ceph-server-2 ceph-server-3
```

We also should make sure the keyring is readable:

```console
ceph-admin$ sudo chmod +r /etc/ceph/ceph.client.admin.keyring
```

Finally, check on the health of the cluster:

```console
ceph-admin$ ceph health
```

It should report a state of `active + clean` once it has finished peering.

Congratulations!

## Expanding the cluster

To more closely model a production cluster, we're going to add one more OSD daemon and a [Ceph Metadata Server](http://ceph.com/docs/master/man/8/ceph-mds/). We'll also add monitors to all hosts instead of just one.

### Add an OSD
```console
$ vagrant ssh ceph-server-1 -c 'sudo mkdir /var/local/osd2'
```

Now, from the admin node, we prepare and activate the OSD:
```console
ceph-admin$ ceph-deploy osd prepare ceph-node-1:/var/local/osd2
ceph-admin$ ceph-deploy osd activate ceph-node-1:/var/local/osd2
```

Watch the rebalancing:

```console
ceph-admin$ ceph -w
```

You should eventually see it return to an `active+clean` state.

### Add metadata server

Let's add a metadata server to server1:

```console
ceph-admin$ ceph-deploy mds create ceph-server-1
```

## Add more monitors

We add monitors to servers 2 and 3.

```console
ceph-admin$ ceph-deploy mon create ceph-server-2 ceph-server-3
```

Watch the quorum status, and ensure it's happy:

```console
ceph-admin$ ceph quorum_status --format json-pretty
```

## Install Ceph Object Gateway

TODO

## Play around!

Now that we have everything set up, let's actually use the cluster. We'll use the ceph-client machine for this.

### Create a block device

TODO

### Create a mount with Ceph FS

TODO

### Store a blob object

TODO

## Cleanup

When you're all done, tell Vagrant to destroy the VMs.

```console
$ vagrant destroy -f
```