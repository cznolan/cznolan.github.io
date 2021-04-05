---
layout: post
title: "Cisco Device Backup With Oxidized"
date: 2021-04-05 19:30:00 +0800
tags: cisco backup oxidized xe xr nx-os ios
author: cznolan
---
I have recently been looking for a reasonable option for backing up network device configurations. It looked like the world had moved away from RANCID toward Oxidized, so I decided to try it out in my lab.

This guide will run through setting up a basic Oxidized instance running as a Docker container, and having it write backup files to a GitHub repository.

My setup is summarised as follows:
* Docker host is Ubuntu 20.04.2 LTS
* Docker version is 20.10.5
* Oxidized version is 0.28.0 from 27 February 2021
* IOS XE device is running 16.12.4a
* IOS XR device is running 7.0.1
* NX-OS device is running 9.3.3

## Docker Host Setup

For my Ubuntu Docker host the only thing I have installed during the Ubuntu setup wizard is OpenSSH server, so I will first upgrade the installed packages.

```
sudo apt-get update
sudo apt-get upgrade
```

To install Docker, you can follow the official Docker installation guide here.
[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/){:target="_blank"}


The abridged version of the Docker installation procedure is that we must first add the Docker repository GPG key.
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

And we can then add the Docker repository.

```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

Now we can install the required Docker packages.

```
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

You can continue to make any changes you wish to the Ubuntu configuration, however there are no other prerequisites for us to proceed with building the Docker image for Oxidized.

## Docker Image Build

To build the Oxidized Docker container, we can very simply follow the instructions on the Oxidized GitHub page.
[https://github.com/ytti/oxidized#running-with-docker](https://github.com/ytti/oxidized#running-with-docker){:target="_blank"}


The basic steps are that we will first clone the Oxidized GitHub repo.

```
git clone https://github.com/ytti/oxidized
```

And subsequently build the Docker container image.

```
sudo docker build -q -t oxidized/oxidized:latest oxidized/
```

A directory then needs to be created to store all the Oxidized configuration in. This directory is mapped to __/root/.config/oxidized__ inside the container.

```
sudo mkdir /etc/oxidized
```

You must then run oxidized once to create the initial configuration files. The container will immediately close and remove itself.

```
sudo docker run --rm -v /etc/oxidized:/root/.config/oxidized -p 8888:8888/tcp -t oxidized/oxidized:latest oxidized
```

## Oxidized Configuration

We will then need to create an inventory file. You can call this file whatever you want, but you must put it somewhere in the _/etc/oxidized_ parent directory for the Docker container to be able to read the file. Other inventory options are available such as SQL database, however for my lab setup a text file is fine.

```
sudo vi /etc/oxidized/router.db
```

For my lab setup, I want to specify a few parameters within the inventory file as I do not have DNS available, and my IOS XR device is configured with different SSH credentials. I have adopted a format of
__hostname:ip_address:model:group__ to be able to specify the IP addresses, as well as a group parameter which can be used to separate my IOS XR device.

The full list of supported models can be found here. Just use the name of the file without the _.rb_ extension.
[https://github.com/ytti/oxidized/tree/master/lib/oxidized/model](https://github.com/ytti/oxidized/tree/master/lib/oxidized/model){:target="_blank"}



My router.db file looks like this.

```
xe-02:192.168.217.232:iosxe:cisco
xr-02:192.168.217.251:iosxr:xr
n9k-01:192.168.217.250:nxos:cisco
```


You will probably need to tweak your inventory format a bit until you find something that works, but for my lab setup we will move on to editing the Oxidized configuration file.

```
sudo vi /etc/oxidized/config
```

There are a lot of options available here, so I am going to summarise what I have configured.

* Default SSH username and password of _admin_
* Backup collection interval is set to 86400 seconds (24 hours)
* Configuration files are output to a local Git repository
* Hooks are used to push the Git repository into a GitHub SaaS repository
* Source file is a colon-delimited "CSV" with four columns mapped
* Dedicated group for IOS XR devices in order to specify different username and password
* As seen in the GitHub SaaS repository, changes are pushed by the _Oxidized-user_ account
* GitHub SaaS Personal Access Token is used for authentication

My configuration file looks like this.

```yaml
---
username: admin
password: admin
resolve_dns: false
interval: 86400
use_syslog: false
debug: true
threads: 30
timeout: 20
retries: 3
prompt: !ruby/regexp /^([\w.@-]+[#>]\s?)$/
rest: 127.0.0.1:8888
next_adds_job: false
vars: {}
models: {}
pid: "~/.config/oxidized/pid"
crash:
  directory: "~/.config/oxidized/crashes"
  hostnames: false
stats:
  history_size: 10
input:
  default: ssh
  debug: false
  ssh:
    secure: false
  ftp:
    passive: true
  utf8_encoded: true
output:
  default: git
  git:
    user: Oxidized-user
    email: o@lab.example
    single_repo: true
    repo: "~/.config/oxidized/git/development.git"
hooks:
  push_to_remote:
    type: githubrepo
    events: [post_store]
    remote_repo: https://github.com/cznolan/development.git
    username: cznolan
    password: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
source:
  default: csv
  csv:
    file: "~/.config/oxidized/router.db"
    delimiter: !ruby/regexp /:/
    map:
      name: 0
      ip: 1
      model: 2
      group: 3
groups:
  xr:
    username: root
    password: admin123
```

If you wanted to just write the configurations to files, you could substitute this output configuration section.

```yaml
output:
  default: file
  file:
    directory: ~/.config/oxidized/configs
```

## Running Oxidized

In order to run Oxidized, we will first create the container and then look up the container ID and name.

```
sudo docker create -v /etc/oxidized:/root/.config/oxidized -p 8888:8888/tcp -t oxidized/oxidized:latest oxidized
```

In my lab, the randomised container name is _competent_gould_.

```
oxidized@oxidized:~$ sudo docker ps -a
CONTAINER ID   IMAGE                      COMMAND      CREATED       STATUS                         PORTS     NAMES
aa52e1dae939   oxidized/oxidized:latest   "oxidized"   2 hours ago   Exited (1) About an hour ago             competent_gould
```

We can then start the container and attach the standard streams (stdin, stdout, stderr) to the container so we can see what is happening.

```
sudo docker start -a competent_gould
```

To detach from the container (but leave it running in the background) you can use _Ctrl+C_.

## Results

The resulting output is that we get a folder for each group inside the GitHub SaaS repository.

![Folder Structure](/images/2021-04-05-oxidized-repo.png)

And for each file in the repository there will be a commit history for each time the configuration has changed.

![Configuration History](/images/2021-04-05-oxidized-history.png)

GitHub SaaS also has the in-built diff tool to compare the files and see what has changed with each committed device configuration file.

## Observations

These are a few observations I have made using Oxidized.

* The _githubrepo_ hook uses the repository _master_ branch. If you are creating a new repository on GitHub for Oxidized backups, just take note that GitHub will now make the default branch _main_ rather than _master_. You can change the default branch in your repository, however it does not appear to be possible to define a branch in Oxidized at this time.

* The _githubrepo_ hook combined with the _post_store_ event pushes each changed file individually. There does not appear to be an option in Oxidized at this time to perform a single push for all changed devices, so you will see some chattiness in the number of commits. I'm not sure whether the _nodes_done_ event would resolve this, as I received error messages when I tried it.

* The username and passwords in the configuration file are echoed in the debug log. You may wish to review and sanitise any debug logs before posting them online for support. You can see this in my log sample below.

* All supported inventory source and backup destination types can be found here
[https://github.com/ytti/oxidized/blob/master/README.md](https://github.com/ytti/oxidized/blob/master/README.md){:target="_blank"}

* Oxidized backup files sanitise the common passwords found on devices, but may not capture all of them. I would suggest reviewing your backups, and if required you can edit the device models. You can see how each model is configured by looking on the Oxidized GitHub page, or by looking at the models within the Docker container.

```
sudo docker exec -it competent_gould /bin/bash
ls /var/lib/gems/2.5.0/gems/oxidized-0.28.0/lib/oxidized/model/
```

## Troubleshooting

These are some troubleshooting steps I had to take with my deployment.

* Make sure you haven't copied my Docker container name _competent_gould_ in your CLI commands anywhere.

* When stopping the Docker container, I was unable to restart it due to Oxidized thinking it was already running. This was resolved by removing the _pid_ file manually. It seems to happen around 50% of the time.

```
sudo rm /etc/oxidized/pid
```

* When changing _githubrepo_ hook configuration in the configuration file, I needed to delete the repo for the changes to take affect.

```
sudo rm -rf /etc/oxidized/git/development.git
```

* Removing the local repository once led me to encounter the following error when trying to use the _githubrepo_ hook.

```
ERROR -- : Hook push_to_remote (#<GithubRepo:0x000055fcfbda2138>) failed (#<Rugged::ReferenceError: cannot push non-fastforwardable reference>) for event :post_store
```

This was resolved by doing a manual push of the local repo into GitHub SaaS.

```
cd /etc/oxidized/git/development.git
sudo git remote add remote-repo https://github.com/cznolan/development.git
sudo git push remote-repo -f
```

## Sample Logs

Below are some sample debug logs from Oxidized when only a single device called xe-02 is configured in the inventory file.

```
oxidized@oxidized:/etc/oxidized$ sudo docker start -a competent_gould
I, [2021-04-05T07:55:56.937224 #1]  INFO -- : Oxidized starting, running as pid 1
D, [2021-04-05T07:55:56.937897 #1] DEBUG -- : Hook "push_to_remote" registered GithubRepo for event :post_store
I, [2021-04-05T07:55:56.938449 #1]  INFO -- : lib/oxidized/nodes.rb: Loading nodes
D, [2021-04-05T07:55:56.938546 #1] DEBUG -- : resolving DNS for xe-02...
D, [2021-04-05T07:55:56.938656 #1] DEBUG -- : IPADDR 192.168.217.232
D, [2021-04-05T07:55:56.938714 #1] DEBUG -- : node.rb: resolving node key 'model', with passed global value of '' and node value 'iosxe'
D, [2021-04-05T07:55:56.938734 #1] DEBUG -- : node.rb: setting node key 'model' to value 'junos' from global
D, [2021-04-05T07:55:56.938807 #1] DEBUG -- : node.rb: returning node key 'model' with value 'iosxe'
D, [2021-04-05T07:55:56.938911 #1] DEBUG -- : lib/oxidized/node.rb: Loading model "iosxe"
D, [2021-04-05T07:55:56.939986 #1] DEBUG -- : lib/oxidized/model/model.rb Added all to the commands list
D, [2021-04-05T07:55:56.940030 #1] DEBUG -- : lib/oxidized/model/model.rb Added secret to the commands list
D, [2021-04-05T07:55:56.940064 #1] DEBUG -- : lib/oxidized/model/model.rb Added show version to the commands list
D, [2021-04-05T07:55:56.940094 #1] DEBUG -- : lib/oxidized/model/model.rb Added show vtp status to the commands list
D, [2021-04-05T07:55:56.940139 #1] DEBUG -- : lib/oxidized/model/model.rb Added show inventory to the commands list
D, [2021-04-05T07:55:56.940168 #1] DEBUG -- : lib/oxidized/model/model.rb Added show running-config to the commands list
D, [2021-04-05T07:55:56.940405 #1] DEBUG -- : node.rb: resolving node key 'input', with passed global value of 'ssh' and node value ''
D, [2021-04-05T07:55:56.940518 #1] DEBUG -- : node.rb: returning node key 'input' with value 'ssh'
D, [2021-04-05T07:55:56.992221 #1] DEBUG -- : node.rb: resolving node key 'output', with passed global value of 'git' and node value ''
D, [2021-04-05T07:55:56.992321 #1] DEBUG -- : node.rb: returning node key 'output' with value 'git'
D, [2021-04-05T07:55:57.004375 #1] DEBUG -- : node.rb: resolving node key 'username', with passed global value of '' and node value ''
D, [2021-04-05T07:55:57.004439 #1] DEBUG -- : node.rb: setting node key 'username' to value 'admin' from global
D, [2021-04-05T07:55:57.004479 #1] DEBUG -- : node.rb: returning node key 'username' with value 'admin'
D, [2021-04-05T07:55:57.004511 #1] DEBUG -- : node.rb: resolving node key 'password', with passed global value of '' and node value ''
D, [2021-04-05T07:55:57.004544 #1] DEBUG -- : node.rb: setting node key 'password' to value 'admin' from global
D, [2021-04-05T07:55:57.004578 #1] DEBUG -- : node.rb: returning node key 'password' with value 'admin'
I, [2021-04-05T07:55:57.004666 #1]  INFO -- : lib/oxidized/nodes.rb: Loaded 1 nodes
D, [2021-04-05T07:55:57.219674 #1] DEBUG -- : lib/oxidized/core.rb: Starting the worker...
Puma starting in single mode...
* Version 3.11.4 (ruby 2.5.1-p57), codename: Love Song
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://127.0.0.1:8888
Use Ctrl-C to stop
D, [2021-04-05T07:55:58.222070 #1] DEBUG -- : lib/oxidized/worker.rb: Jobs running: 0 of 1 - ended: 0 of 1
D, [2021-04-05T07:55:58.222353 #1] DEBUG -- : lib/oxidized/worker.rb: Added cisco/xe-02 to the job queue
D, [2021-04-05T07:55:58.222388 #1] DEBUG -- : lib/oxidized/worker.rb: 1 jobs running in parallel
D, [2021-04-05T07:55:58.222748 #1] DEBUG -- : lib/oxidized/job.rb: Starting fetching process for xe-02 at 2021-04-05 07:55:58 UTC
D, [2021-04-05T07:55:58.222926 #1] DEBUG -- : lib/oxidized/input/ssh.rb: Connecting to xe-02
D, [2021-04-05T07:55:58.223213 #1] DEBUG -- : AUTH METHODS::["none", "publickey", "password"]
D, [2021-04-05T07:55:58.670254 #1] DEBUG -- : lib/oxidized/input/ssh.rb: expecting [/^([\w.@()-]+[#>]\s?)$/] at xe-02
D, [2021-04-05T07:55:59.223120 #1] DEBUG -- : lib/oxidized/worker.rb: 1 jobs running in parallel
D, [2021-04-05T07:55:59.474736 #1] DEBUG -- : lib/oxidized/input/cli.rb: Running post_login commands at xe-02
D, [2021-04-05T07:55:59.474831 #1] DEBUG -- : lib/oxidized/input/cli.rb: Running post_login command: nil, block: #<Proc:0x000055b03bf642d0@/var/lib/gems/2.5.0/gems/oxidized-0.28.0/lib/oxidized/model/ios.rb:127> at xe-02
D, [2021-04-05T07:55:59.475195 #1] DEBUG -- : lib/oxidized/input/cli.rb: Running post_login command: "terminal length 0", block: nil at xe-02
D, [2021-04-05T07:55:59.475243 #1] DEBUG -- : lib/oxidized/input/ssh.rb terminal length 0 @ xe-02 with expect: /^([\w.@()-]+[#>]\s?)$/
D, [2021-04-05T07:55:59.475356 #1] DEBUG -- : lib/oxidized/input/ssh.rb: expecting [/^([\w.@()-]+[#>]\s?)$/] at xe-02
D, [2021-04-05T07:55:59.878354 #1] DEBUG -- : lib/oxidized/input/cli.rb: Running post_login command: "terminal width 0", block: nil at xe-02
D, [2021-04-05T07:55:59.878425 #1] DEBUG -- : lib/oxidized/input/ssh.rb terminal width 0 @ xe-02 with expect: /^([\w.@()-]+[#>]\s?)$/
D, [2021-04-05T07:55:59.878635 #1] DEBUG -- : lib/oxidized/input/ssh.rb: expecting [/^([\w.@()-]+[#>]\s?)$/] at xe-02
D, [2021-04-05T07:56:00.223371 #1] DEBUG -- : lib/oxidized/worker.rb: 1 jobs running in parallel
D, [2021-04-05T07:56:00.281869 #1] DEBUG -- : lib/oxidized/model/model.rb Collecting commands' outputs
D, [2021-04-05T07:56:00.281960 #1] DEBUG -- : lib/oxidized/model/model.rb Executing show version
D, [2021-04-05T07:56:00.281996 #1] DEBUG -- : lib/oxidized/input/ssh.rb show version @ xe-02 with expect: /^([\w.@()-]+[#>]\s?)$/
D, [2021-04-05T07:56:00.282403 #1] DEBUG -- : lib/oxidized/input/ssh.rb: expecting [/^([\w.@()-]+[#>]\s?)$/] at xe-02
D, [2021-04-05T07:56:00.685531 #1] DEBUG -- : lib/oxidized/model/model.rb Executing show vtp status
D, [2021-04-05T07:56:00.685608 #1] DEBUG -- : lib/oxidized/input/ssh.rb show vtp status @ xe-02 with expect: /^([\w.@()-]+[#>]\s?)$/
D, [2021-04-05T07:56:00.685983 #1] DEBUG -- : lib/oxidized/input/ssh.rb: expecting [/^([\w.@()-]+[#>]\s?)$/] at xe-02
D, [2021-04-05T07:56:01.089097 #1] DEBUG -- : lib/oxidized/model/model.rb Executing show inventory
D, [2021-04-05T07:56:01.089172 #1] DEBUG -- : lib/oxidized/input/ssh.rb show inventory @ xe-02 with expect: /^([\w.@()-]+[#>]\s?)$/
D, [2021-04-05T07:56:01.089384 #1] DEBUG -- : lib/oxidized/input/ssh.rb: expecting [/^([\w.@()-]+[#>]\s?)$/] at xe-02
D, [2021-04-05T07:56:01.223938 #1] DEBUG -- : lib/oxidized/worker.rb: 1 jobs running in parallel
D, [2021-04-05T07:56:01.492187 #1] DEBUG -- : lib/oxidized/model/model.rb Executing show running-config
D, [2021-04-05T07:56:01.492263 #1] DEBUG -- : lib/oxidized/input/ssh.rb show running-config @ xe-02 with expect: /^([\w.@()-]+[#>]\s?)$/
D, [2021-04-05T07:56:01.492574 #1] DEBUG -- : lib/oxidized/input/ssh.rb: expecting [/^([\w.@()-]+[#>]\s?)$/] at xe-02
D, [2021-04-05T07:56:01.897269 #1] DEBUG -- : lib/oxidized/input/cli.rb Running pre_logout commands at xe-02
D, [2021-04-05T07:56:01.897342 #1] DEBUG -- : lib/oxidized/input/ssh.rb exit @ xe-02 with expect: nil
D, [2021-04-05T07:56:02.003912 #1] DEBUG -- : lib/oxidized/node.rb: Oxidized::SSH ran for xe-02 successfully
D, [2021-04-05T07:56:02.004036 #1] DEBUG -- : lib/oxidized/job.rb: Config fetched for xe-02 at 2021-04-05 07:56:02 UTC
I, [2021-04-05T07:56:02.226750 #1]  INFO -- : Configuration updated for cisco/xe-02
I, [2021-04-05T07:56:02.226957 #1]  INFO -- : GithubRepo: Pushing local repository(/root/.config/oxidized/git/development.git/)...
I, [2021-04-05T07:56:02.227135 #1]  INFO -- : GithubRepo: to remote: https://github.com/cznolan/development.git
D, [2021-04-05T07:56:02.712099 #1] DEBUG -- : GithubRepo: Authenticating using username and password as 'cznolan'
D, [2021-04-05T07:56:03.031320 #1] DEBUG -- : GithubRepo: {:total_objects=>0, :indexed_objects=>0, :received_objects=>0, :local_objects=>0, :total_deltas=>0, :indexed_deltas=>0, :received_bytes=>0}
D, [2021-04-05T07:56:03.031395 #1] DEBUG -- : GithubRepo: nothing received after fetch
D, [2021-04-05T07:56:03.516272 #1] DEBUG -- : GithubRepo: Authenticating using username and password as 'cznolan'
D, [2021-04-05T07:56:05.371354 #1] DEBUG -- : lib/oxidized/worker.rb: Jobs running: 0 of 1 - ended: 1 of 1
D, [2021-04-05T07:56:05.371447 #1] DEBUG -- : lib/oxidized/worker.rb: Running :nodes_done hook
D, [2021-04-05T07:56:06.371798 #1] DEBUG -- : lib/oxidized/worker.rb: Jobs running: 0 of 1 - ended: 0 of 1
D, [2021-04-05T07:56:07.372833 #1] DEBUG -- : lib/oxidized/worker.rb: Jobs running: 0 of 1 - ended: 0 of 1
D, [2021-04-05T07:56:08.373530 #1] DEBUG -- : lib/oxidized/worker.rb: Jobs running: 0 of 1 - ended: 0 of 1
D, [2021-04-05T07:56:09.373955 #1] DEBUG -- : lib/oxidized/worker.rb: Jobs running: 0 of 1 - ended: 0 of 1
D, [2021-04-05T07:56:10.374598 #1] DEBUG -- : lib/oxidized/worker.rb: Jobs running: 0 of 1 - ended: 0 of 1
```
