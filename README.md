
# Local Docker + Ceph + Flocker Cluster with Vagrant

### What you will need

- Vagrant
- VirtualBox
- Git
- Ansible

This repo has been tested on
- Vagrant 1.7.4
- VirtualBox Version 5.0.16 r105871
- Ansible 2.0.1.0
- Mac OSX El Capitan 10.11.2

## How to use this repository

This repository will set up 4 local virtualbox VMs. You will need 2 to 4 GB of memory.

1. RadosGW, MDS, Flocker Control
2. OSD, MON, Flocker Agent + Docker Plugin
3. OSD, MON, Flocker Agent + Docker Plugin
4. OSD, MON, Flocker Agent + Docker Plugin

Getting started, clone this repo, install some tools.

Install a few things.
If you have any errors with installing `flocker-ca` see our [documentation](https://docs.clusterhq.com/en/latest/flocker-standalone/install-client.html) on installing the flocker client.
```
brew install flocker-1.11.0   # (this lets us use `flocker-ca`)
brew install ansible          # (or you can install ansible inside a python virtualenv.)
vagrant plugin install vai
```

Clone this Repository, `cd` to it. 
> note make sure to use `--recursive` to get the ceph-ansible submodule

```
git clone --recursive https://github.com/ClusterHQ/flocker-ceph-vagrant
cd flocker-ceph-vagrant
```

Next, clone ceph-ansible.
```
cd ceph-ansible
```

Then, copy some pre-baked configuration provided as part of this repo.
```
../ready_env.sh
```

> Note: the next command will take ~10-12 minutes to complete. It will take longer if its your first time running it because you will need to download the Vagrant `.box`. It will also ask you for your password in order to modify `/etc/hosts` with convenient aliases to play around with your cluster.

Create and Provision everything
```
vagrant up --provider=virtualbox
```

An inventory is used, and should be written to `ansible/inventory`
```
$ cat ansible/inventory/vagrant_ansible_inventory
# Generated by Vagrant

ceph1 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2222 ansible_ssh_private_key_file=/Users/<USER>/Desktop/path-to/flocker-ceph-vagrant/ceph-ansible/.vagrant/machines/ceph1/virtualbox/private_key ansible_ssh_user=vagrant
ceph2 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2200 ansible_ssh_private_key_file=/Users/<USER>/Desktop/path-to/flocker-ceph-vagrant/ceph-ansible/.vagrant/machines/ceph2/virtualbox/private_key ansible_ssh_user=vagrant
ceph3 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_private_key_file=/Users/<USER>/Desktop/path-to/flocker-ceph-vagrant/ceph-ansible/.vagrant/machines/ceph3/virtualbox/private_key ansible_ssh_user=vagrant
ceph4 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2202 ansible_ssh_private_key_file=/Users/<USER>/Desktop/path-to/flocker-ceph-vagrant/ceph-ansible/.vagrant/machines/ceph4/virtualbox/private_key ansible_ssh_user=vagrant

[osds]
ceph2
ceph3
ceph4

[mons]
ceph2
ceph3
ceph4

[mdss]
ceph1

[rdgws]
ceph1

[flocker_agents]
ceph2
ceph3
ceph4

[flocker_control_service]
ceph1

[flocker_docker_plugin]
ceph2
ceph3
ceph4

[flocker_ceph]
ceph2
ceph3
ceph4

[nodes:children]
flocker_agents
flocker_control_service
flocker_docker_plugin
flocker_ceph
```

Also in either case, once complete, a healthy ceph cluster should exist
```
vagrant ssh ceph2 -c "sudo ceph -s"
    cluster 4a158d27-f750-41d5-9e7f-26ce4c9d2d45
     health HEALTH_OK
     monmap e1: 3 mons at {ceph2=192.168.5.3:6789/0,ceph3=192.168.5.4:6789/0,ceph4=192.168.5.5:6789/0}
            election epoch 4, quorum 0,1,2 ceph2,ceph3,ceph4
     mdsmap e6: 1/1/1 up {0=ceph1=up:active}
     osdmap e20: 6 osds: 6 up, 6 in
            flags sortbitwise
      pgmap v30: 320 pgs, 3 pools, 1960 bytes data, 20 objects
            221 MB used, 5887 MB / 6109 MB avail
                 320 active+clean
  client io 2030 B/s wr, 17 op/s
```

Check your Flocker Cluster
```
$ vagrant ssh ceph1 -c "sudo curl --cacert /etc/flocker/cluster.crt \
   --cert /etc/flocker/plugin.crt \
   --key /etc/flocker/plugin.key \
   --header 'Content-type: application/json' \
   https://ceph1:4523/v1/state/nodes | python -m json.tool"
[
    {
        "host": "192.168.5.5",
        "uuid": "3aab9ca0-a48e-4beb-8f8c-4e814f8cecf8"
    },
    {
        "host": "192.168.5.3",
        "uuid": "2a00f5c9-b843-41c7-adf4-99013d09a594"
    },
    {
        "host": "192.168.5.4",
        "uuid": "ac3cc217-7706-447b-b1ff-a0c774d95c6b"
    }
]
```

Use your Flocker Cluster
```
$ vagrant ssh ceph3 -c "sudo docker volume create -d flocker --name test -o size=10G"
(The mountpoint may take ~10s to show up if you run this command quckly after the above)
$ vagrant ssh ceph3 -c "sudo df -h | grep flocker"
/dev/rbd1       9.8G   23M  9.2G   1% /flocker/2879f72f-680a-404c-8610-f1f9d87cc1f1
```

```
$ vagrant ssh ceph3 -c "sudo docker run --volume-driver flocker \
   -v test:/data --name test-container -itd busybox"
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
385e281300cc: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:4a887a2326ec9e0fa90cce7b4764b0e627b5d6afcb81a3f73c85dc29cea00048
Status: Downloaded newer image for busybox:latest
d943aa00250c551bb0b84f9eb31c03662369e5ff8fa69f070b78941fa80e4640
```

```
$ vagrant ssh ceph3 -c "sudo docker ps"
CONTAINER ID  IMAGE    COMMAND  CREATED        STATUS       PORTS   NAMES
d943aa00250c  busybox  "sh"     24 seconds ago Up 21 seconds        test-container
```

```
$ vagrant ssh ceph3 -c "sudo docker inspect -f "{{.Mounts}}" test-container"
[{test /flocker/83a09e31-f6a9-478e-8e7b-53b978f79c21 /data flocker  true rprivate}]
```

```
$ vagrant ssh ceph1 -c "sudo curl --cacert /etc/flocker/cluster.crt \
   --cert /etc/flocker/plugin.crt \
   --key /etc/flocker/plugin.key \
   --header 'Content-type: application/json' \
   https://ceph1:4523/v1/state/datasets | python -m json.tool"
[
    {
        "dataset_id": "83a09e31-f6a9-478e-8e7b-53b978f79c21",
        "maximum_size": 10737418240,
        "path": "/flocker/83a09e31-f6a9-478e-8e7b-53b978f79c21",
        "primary": "5f4be886-3cf7-434b-975f-5babeea63a63"
    }
]
```

```
$ vagrant ssh ceph1 -c "sudo curl --cacert /etc/flocker/cluster.crt \
   --cert /etc/flocker/plugin.crt \
   --key /etc/flocker/plugin.key \
   --header 'Content-type: application/json' \
   https://ceph1:4523/v1/configuration/datasets | python -m json.tool"
[
    {
        "dataset_id": "83a09e31-f6a9-478e-8e7b-53b978f79c21",
        "deleted": false,
        "maximum_size": 10737418240,
        "metadata": {
            "maximum_size": "10737418240",
            "name": "test"
        },
        "primary": "5f4be886-3cf7-434b-975f-5babeea63a63"
    }
]
```

Move the volume from `ceph3` to `ceph4`
```
$ vagrant ssh ceph3 -c "sudo docker rm -f test-container"
$ vagrant ssh ceph4 -c "sudo docker run --volume-driver flocker \
   -v test:/data --name test-container -itd busybox"
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
385e281300cc: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:4a887a2326ec9e0fa90cce7b4764b0e627b5d6afcb81a3f73c85dc29cea00048
Status: Downloaded newer image for busybox:latest
9040ffcc96f704206ccb1e71a354494dfab7a4d25e72c87a4c88bbeefdfdf85d
```

Check its own the correct host now
```
# Not on ceph3 anymore
$ vagrant ssh ceph3 -c "sudo docker ps"
$ vagrant ssh ceph3 -c "sudo df -h | grep flocker"

# Now its present on ceph4
$ vagrant ssh ceph4 -c "sudo docker ps"
CONTAINER ID  IMAGE    COMMAND  CREATED        STATUS       PORTS   NAMES
9040ffcc96f7  busybox  "sh"     24 seconds ago Up 21 seconds        test-container

$ vagrant ssh ceph4 -c "sudo df -h | grep flocker"
/dev/rbd1       9.8G   23M  9.2G   1% /flocker/83a09e31-f6a9-478e-8e7b-53b978f79c21
```

To list your volumes in Ceph use the below command.
```
$ vagrant ssh ceph2 -c "sudo  rbd ls"
flocker-83a09e31-f6a9-478e-8e7b-53b978f79c21
flocker-d2bfb016-e981-4f87-827b-9af6c0575ba2
```

# Install and use Swarm

Optionally there are a few scripts you can run to get Swarm running on your small cluster.

```
../scripts/ready-docker-for-swarm.sh
../scripts/install-docker-swarm.sh
.
.
.
Done: Swarm available at tcp://192.168.5.2:3375
```

> Note, in order to talk to swarm you need to have the docker client available on your host machine.
> If you do not, visit [Docker ToolBox](https://www.docker.com/products/docker-toolbox) to install it.

To use your new Swarm cluster, run the following.
```
export DOCKER_HOST=tcp://192.168.5.2:3375
unset DOCKER_TLS_VERIFY
docker info
```

You should see output from swarm. Then, you can run containers against Swarm just like any other Docker daemon.
```
$ docker run -d --name=redis-server  --volume-driver=flocker -v testfailover:/data --restart=always -e reschedule:on-node-failure  redis redis-server --appendonly yes
66c0882809aaf1078f75c57433b85d2eadcebd35370bb16e360b57163e68c777
```

Notice the `NAMES` now has which ceph host the container is running on.
```
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
66c0882809aa        redis               "docker-entrypoint.sh"   26 seconds ago      Up 1 seconds        6379/tcp            ceph3/redis-server
```

Verify that the Redis server is actually using a Flocker volume.
```
$ docker inspect -f "{{.Mounts}}"  redis-server
[{testfailover /flocker/ba263f1f-4ed5-440b-8450-6a5cc632ad2c /data flocker  true rprivate}]
```

## Cleanup

To clean up and delete your cluster run the following
```
$ vagrant destroy -f
==> ceph4: Forcing shutdown of VM...
==> ceph4: Destroying VM and associated drives...
==> ceph4: Running cleanup tasks for 'shell' provisioner...
==> ceph4: Running cleanup tasks for 'vai' provisioner...
==> ceph4: Running cleanup tasks for 'ansible' provisioner...
==> ceph3: Forcing shutdown of VM...
==> ceph3: Destroying VM and associated drives...
==> ceph3: Running cleanup tasks for 'shell' provisioner...
==> ceph2: Forcing shutdown of VM...
==> ceph2: Destroying VM and associated drives...
==> ceph2: Running cleanup tasks for 'shell' provisioner...
==> ceph1: Forcing shutdown of VM...
==> ceph1: Destroying VM and associated drives...
==> ceph1: Running cleanup tasks for 'shell' provisioner...
```

## More information

In case you are curious , running behind the scenes inside vagrant is ansible and you can re run it after `vagrant up` for re-runs because we have output the ansible inventory to your local machine.
```
ansible-playbook -i ansible/inventory/vagrant_ansible_inventory site.yml \
   --extra-vars "fsid=4a158d27-f750-41d5-9e7f-26ce4c9d2d45 \
   monitor_secret=AQAWqilTCDh7CBAAawXt6kyTgLFCxSvJhTEmuw== \
   flocker_agent_yml_path=${PWD}/../agent.yml"
```

## Thanks

The below resources helped us get this going.

- https://www.vagrantup.com/docs/provisioning/ansible_intro.html
- https://github.com/ceph/ceph-ansible 
- https://github.com/ClusterHQ/ansible-role-flocker/
- https://github.com/ClusterHQ/ceph-flocker-driver
- https://github.com/MatthewMi11er/vai
- https://github.com/ceph/ceph-ansible/issues/136

## License 

MIT / BSD
