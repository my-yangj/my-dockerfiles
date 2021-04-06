# my dockerfiles

## overview
This repository contains scripts to setup a dev-ops environment based on freeipa and docker. It targets a small cluster of computer nodes,
though a single node works fine. The env contains a management node and a number of computing nodes. (The management node works as a computer node
in a single node env. )

The management node provides services of networking/storage, user authentication, job-distribution and etc.  And the computing nodes provide the computing resource 
based on docker/podman.

the below is the key software/technology being used, and ubuntu 20.04 is used as the base OS for both management node and computing nodes.
(other linux should works but not tested; clearlinux is a good candidate which provides more performance, and server-friendly features: e.g. controlled os update/mixer,
 reference setup of software stack; iPXE; and much more) 
- linux/lvm2/nfs
- docker-compose for mysql/gitea/jenkins/freeipa
- podman/docker/vagrant run on computing nodes

## dockerfiles: 
Dockerfiles below are used to generate local maintained container images, if image with similar functionality available from public docker image registry, the local
image could be replaced to ease the environment maintaining effort. 

- centosDockerfile: 
this container run on terminal servers. it mounts the data-volume and creates vnc for user to connect. it is selected if a user dont like the default vnc-server on the terminal server directly, or a user want to mounts the persistent-volumes of proj-data and tools to specific directories. Users can create container using podman, and connect into it through vnc. Alternatively, a user could start vnc service on terminal server directly if the default work env works for them (such as gui, default soft).  
      ```bash
        docker run --volume /data:/data:rw -v /tmp/.X11-unix:/tmp/.X11-unix --env DISPLAY=unix$DISPLAY -it ubuntu bash
      ```
- users access computing resources through terminal services
Terminal servers in intranet are just normal worknode.  While terminal servers to be connected from internet are DMZ computers. They have two network interfaces, with one connected with cluster and another open to outside. users from outside of intranet can only connect to DMZ servers which have firewall/port-forwarding enforced e.g.
   - external nic has public ip and domain name, and user connected to these external nic
   - vpn supported through wireguard, which has its own subnet
   - user connections are forwarded to internal nic e.g. access vnc-server run on cluster from wireguard nic controlled by iptables.

- freeipaDockerfile:  (official freeipa/freeipa-server:centos-8 works, thus the local dockerfile is retired. docker is preferred instead of vagrant)
I got its dns conflict with ubuntu host, thus dns is not in includedt. for official docker image see dockerhub freeipa/freeipa-server:centos-? and https://github.com/freeipa/freeipa-container. 
  - freeipa provides directory/authentication service,
  - vagrant uses libvirt which save the vm images at /var/lib/libvirt/images/
  - vagrant steps
    ```
    #do below step after install vagrant and pull some vagrant box, ref to 
    useradd -aG libvirt && su -
    mkdir freeipa && cd freeipa
    vagrant init
    #change Vagrantfile and add 'config.vm.network :public_network, :dev=>"virbr0", :mode=>"bridge", :type=>"bridge"'
    #change Vagrantfile content takes effect for new vm, for vm being created, it need destroy and recreate
    vagrant destroy && vagrant init && vagrant up && vagrant ssh  
    #install freeipa, ref to
    ```

- jenkinsDockerfile: (official images works and configured through docker-compose file).
(start  
provide jenkins service; jenkins can use docker daemon at a node thus can distribute workload to cluster. Jenkins also supports distribute workload with ssh (ssh agent).
   ```bash
     docker run -d \
    -v <your_jenkins_home_in_host>:/var/jenkins_home \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /usr/local/bin/docker:/bin/docker \
    -p 8080:8080 -p 50000:50000 \
    --name <my_jenkins> jenkins/jenkins:lts
  ```

- mysqlDockerfile
mysql is required by gitea see: https://github.com/docker-library/mysql/tree/master/8.0

- giteaDockerfile
  gitea see: https://docs.gitea.io/en-us/install-with-docker/
  gitea backup/restore see: https://gist.github.com/sinbad/4bb771b916fa8facaf340af3fc49ee43
  also ref https://github.com/go-gitea/gitea/tree/master/docker
  
## using podman for non-privilege user
master node has the pre-installed tools exports to work nodes with nfs. when a user wants extra tool or need different work enviornment (all nodes are ubuntu20.04), they could create rootless container with podman, which supporst rootless container which is non-privilege.  it specifies the image location as below,  and image location must be host disk *instead of nfs*. otherwise container/images are under user's home.
  ```toml
    [storage]
    driver = "overlay"
    runroot = "/mnt/backup/"
    graphroot = "/mnt/backup/"
    rootless_storage_path = "/mnt/backup/"
  ```
- skopeo can used to inspect the image, manipunate its storage
  
## dnsmasq: to support dhcp and pxe and automatically deploy the computing node (-tbd-) 
  (it is desired to wake up the backup server; start it with hosted image e.g. run a backup and log a report; and then power-off/suspend the server).
  keep the below as reference, but not implemented in my experiment. 
  https://stackoverflow.com/questions/38816077/run-dnsmasq-as-dhcp-server-from-inside-a-docker-container
  pxe env build:  https://netboot.xyz/selfhosting/
  pxe selfhost container: https://github.com/linuxserver/docker-netbootxyz
  pxe w/ dhcp: https://github.com/samdbmg/dhcp-netboot.xyz https://github.com/kmanna/docker-nat-router
  
# setup
1. master node setup see: [host_setup.md](host_setup.md)
  - tools installed on host
    - docker install w/ docker-volumn plugin
    - podman
    - site tools 
    - conda
  - docker_compose file
    - docker compose can start as daemon, the generated image has special signiture so that it doesn't rebuild the image each time.
    
2. compute node setup,
  - docker/podman install w/ nfs volume
  - nextflow 
    local job is dispatched with nextflow pipeline (could use podman rootless) on the node. (or docker swarm to the cluster directly?)
  - compute nodes take workload from jenkins
    
3. work flow
  - user connects to the terminal server
  - user checkout project/tool data, which will mount as container volumes ( common dir are nfs exported from master node)
  - user runs their job through nf pipeline, nf pipeline creates more containers. (user debug the pipeline locally).
  - user submits their job/pipeline into jenkins which delivers jobs to cluster
