---
layout: post
title: "Development environments with Vagrant and Docker"
date: 2014-12-29 22:57
comments: true
categories: 
---
## Facing the problem.

### Repeatable and maintainable development environments

One challenge we have to face when building an application is that the development environment should be an exact copy
of the production environment. This can be easy if we only have to deal with only a single application. Or we are the
only developer building the web application.

In that case, the easiest way to prepare the development environment is to install all the needed packages and
dependencies into the local machine.

Otherwise if we are in a team or we are required to deal with several applications, using the local machine as
development environment is simply not feasible, just because the different applications have different runtime
dependencies or having every developer of the team build and prepare his own development environment is error prone and
could lead to inconsistencies between every developer environment.

For this reason and in order to build development environments repeatable and maintainable across applications and team
members, **[Vagrant](http://vagrantup.com)** takes the stage and allow the configuration of virtual machines as development
environments easily with a custom DSL built on top of Ruby.

### Disclaimer

This article is not about how to get started with Vagrant. It assumes a basic knowledge of how Vagrant works. If you
don't have ever get in touch with Vagrant, I suggest to read the *[Getting Started Vagrant guide](https://docs.vagrantup.com/v2/getting-started/index.html)*
before continue reading the rest of the entry.

## Using vagrant

The solution to the problem could have been to use Vagrant to handle the creation of development environments.

```ruby
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "puphpet/debian75-x64"

  config.vm.provision "shell", inline:
    "echo 'Hi world!'"
end
```

I think this could be the minimum ```Vagrantfile``` that a project could have. Generally applications that build its
development environment using Vagrant have slightly more complex provisioning setups that could use *[Ansible](http://www.ansible.com/)*,
*[Puppet](http://puppetlabs.com/)*, *[Chef](https://www.chef.io/)* or some other *IT automation system*. Currently,
Vagrant supports all of the mentioned and [many many more](https://docs.vagrantup.com/v2/provisioning/index.html).

## Tradeoffs, tradeoffs, tradeoffs ...

This is a good approach to maintain the creation of development environments. So every time we need to bootstrap any
development environment it's as easy as

```bash
$ vagrant up
```

And vagrant brings a virtual machine with all of the application's runtime set up properly to start (or continue)
working in the project. And yes, you've read it well. Every time we need a new development environment, vagrant creates 
one or more new virtual machines if not have been yet created.

So if we have a slightly more complex architecture in production, it means probably that we will need several virtual
machines to be able to handle this. For example, a typical web scenario could be to have some sort of CDN (*Akamai*,
*Amazon CloudFront*, etc.) to handle all the static content of the site. If it doesn't have the static asset, it asks
for it to our web infrastructure. Generally, this infrastructure consists of an nginx or cluster of nginx that *talks*
with the CDN.

This is just an example, but I would not catalog it as *complex*. It's a common example of a web scenario. Another example,
could be if we have some sort of intermediate caching layer. I think of a Varnish for example. Or if we have some
operations in the business logic that performs something in memory with some *Redis* instance.

How do we face the creation of all those services? We could create a virtual machine for every service for example. Having a
dedicated virtual machine for every single service just to mimic our production infrastructure, would be the ideal case but I
think it's not possible. Just because, unless we have *truly monsters* as local host machines, the resources of them are
limited.

So one possible solution to this problem, is to build a single vagrant machine to provide all the services that the
application needs. Although this is a perfectly fine solution to the problem, we can come with another lightweight
solution by using **Docker**.

## Introducing docker

Unless you live secluded in a dark cave, you probably have heard about *[Docker](https://www.docker.com)*. In short,
Docker is a client/server application that is capable to run *processes in an isolated way, by running each one in an
isolated container*. That is, Docker is just another virtualization platform that make use of *[LXC](https://linuxcontainers.org/)*
to provide virtualization at process level rather than in entire machine level.

It's out of the scope of the entry to provide a *[Getting started with Docker](https://docs.docker.com/userguide/)*. In
fact, It assumes that the reader has a **basic** knowledge of how Docker works. So if you don't have tried Docker,
I suggest you to stop here and **[take](https://www.docker.com/tryit/)** the tutorial that Docker's website provides.

### Docker limitations

*(NOTE: If you are not using OS X you can skip ahead this section and go directly to [Docker to the rescue](#docker-to-the-rescue))*

The primarily limitation I've found when using Docker for building development environments, is that OS X (I'm currently
using a Mac Book Pro) does not have native support for *LXC*. **So Docker cannot run natively in OS X**.

But this is not a major real problem, because the Docker guys have already thought of this and the docker client can
be configured to run the Docker commands to the Docker server we want. They have created a project called
**[boot2docker](http://boot2docker.io/)**. ```boot2docker``` is a command-line utility that spawns a lightweight
virtual machine through VirtualBox, that supports natively *LXC* and can run Docker commands.

But, although *boot2docker* is an excellent project that allow us to spawn Docker containers in hosts which don't support
*LXC*, it has from my point of view a major limitation that makes it unable to be used in the development of large applications:
**It only supports file sharing through VirtualBox Shared Folders**. That is, it does not have support for **NFS** and VirtualBox
SharedFolder are **[terrible slow with a large number of files](http://mitchellh.com/comparing-filesystem-performance-in-virtual-machines)**.

So if you have an application with a large number of dynamic includes (that is applications written in an interpreted
language: PHP, Ruby, Python, etc.) I don't recommend the usage of the virtual machine that provides ```boot2docker```.
Maybe in the future they will address this major problem. We, all of the web developers that work with large
applications, hope so :)

Fortunately, the usage of the virtual machine spawned by ```boot2docker``` **is not mandatory**. We can use another
custom virtual machine that could operate directly with ```boot2docker```.

## <a name="docker-to-the-rescue"></a> Docker to the rescue

With Docker the schema changes a bit. Instead of having a virtual machine with all the services running inside or
having several virtual machines one per service to allow them to run in an isolated way, we can bootstrap a single
virtual machine and have all the processes that the application need to run, in an isolated way.

### Setting up a VM to operate with Docker (OS X)

Again **this only applies if you're under OS X**. If you are under some distribution with native support for LXC you will
probably don't have to worry about all of this.

*We can use Vagrant to spawn a virtual machine that acts as if it was provided by boot2docker, but without all the
limitations of boot2docker.* That means, we can spawn a new virtual machine with **NFS support** that will be able to
create Docker containers. In fact, someone has already created a Vagrant box for that:
[https://vagrantcloud.com/yungsang/boxes/boot2docker](https://vagrantcloud.com/yungsang/boxes/boot2docker). This is an
improved version of ```boot2docker``` with some extra benfits but the primary, at least for me, is the support for
NFS shared folders.

To make the point clear **this does not mean that docker volumes and data-only volume containers are mounted through NFS**,
this means that we can mount NFS shared folders on that machine and then *make use of that folders to build volumes and
data-only volume containers*. So we still have to manually mount the shared folders on the machine, but this drawback with
Vagrant is totally painless and can be assumed with almost no cost.

So we could have a Vagrantfile similar to

```ruby
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.box = "yungsang/boot2docker"

    # Synced folders
    config.vm.synced_folder "path/to/a/folder", "/mounted/folder", type: "nfs"

    config.vm.provider "virtualbox" do |virtualbox|
      virtualbox.memory = 2048
    end
end
```

This is a very basic version of the Vagrantfile. With this we only have to bring up the VM and tell ```boot2docker``` that
it must operate with this instance.

```bash
$ vagrant up
$ export DOCKER_HOST=tcp://localhost:2375
```

And to check that everything works properly, the following commands should not fail

```bash
$ docker version
$ docker images
$ docker ps -a
```

With this you are ready to start working with Docker. You're able at this point to start building images and creating
isolated containers that could use the synced folders with an acceptable performance.

## Development environments with Vagrant + Docker

To be fair and if you don't like Vagrant, there are other alternatives: the **[fig](http://www.fig.sh/)** project. Again
if you're under OS X, you will probably have to spawn a VM to save the performance issues with *boot2docker*, so in the
end you'll have to use an intermediate custom VM either with Vagrant or not.

So now, you're truly ready to start building the development environment with **Vagrant** and **Docker**. First of all
we should prepare all the Docker files in order to build the Docker images. For the example we will use

* A data-only volume container that will contain all the source code.
* A container that will run nginx.
* A container that will run php-fpm.

So a possible application layout could be one as follows

```
.
├── src
├── public
│   └── index.php
├── config
│   └── environment
│       └── docker
│           ├── phpfpm
│           │   ├── www.conf
│           │   └── Dockerfile
│           └── nginx
│               ├── vhost.conf
│               ├── start.sh
│               └── Dockerfile
└── Vagrantfile
```

Next we should restructure the Vagrantfile in order to make it more maintainable, and also share the application code as
a NFS mount to achieve an acceptable performance

```ruby
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.define "default", autostart: false do |default|
        default.vm.box = "yungsang/boot2docker"

        # The application code
        default.vm.synced_folder ".", "/var/www", type: "nfs"

        default.vm.provider "virtualbox" do |virtualbox|
            virtualbox.memory = 2048
        end

        default.vm.provision "docker" do |docker|
            docker.build_image "/var/www/config/environment/docker/nginx", args: '-t "nginx"'
            docker.build_image "/var/www/config/environment/docker/php-fpm", args: '-t "phpfpm"'
        end

        # The default machine already has support for SSH
        default.ssh.insert_key = false
        default.ssh.username = "docker"
        default.ssh.password = "tcuser"
    end
end
```

Things to note here:

* With the instruction ```default.vm.synced_folder ".", "/var/www", type: "nfs"``` we have mounted through NFS the
  application source code.
* Then we specified a ```docker``` provisioner for the ```default``` machine that will build all the needed Docker
  images (This has been done that way, because I was not able to make Vagrant to build the images automatically) with
  the names: ```nginx``` and ```phpfpm```.

To end with the ```Vagrantfile```, the only needed thing is to specify the Docker containers as if they were common
Vagrant VMs. First, **in the same Vagrantfile where we defined the ```default``` machine**, let's define the ```app```
Docker container, a *data-only volume container* that will host the application source code

```ruby
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.define "app" do |app|
        app.vm.provider "docker" do |docker|
            docker.image = "debian"
            docker.name = "app"
            docker.volumes = ["/var/www:/var/www"]
            docker.vagrant_vagrantfile = __FILE__
            docker.remains_running = false
        end
    end
end
```

Since the only purpose of this container is to host the application source code, we don't need a ```Dockerfile``` to
build an image for it. So we only need to configure the base image through ```docker.image = "debian"```.
Next we give the name ```app``` to the image, add a data volume with ```docker.volumes = ["/var/www:/var/www"]``` and
tell Vagrant that this container won't be permanently running with ```docker.remains_running = false```. Note that the
data-volumes, have a direct correspondence with the synced folders of the ```default``` machine.

Additionally, Vagrant needs to know which VM should spawn through ```docker.vagrant_vagrantfile = __FILE__```. With
this we are telling Vagrant that this container will be running on the ```default``` machine in the **current**
Vagrantfile. If you're in the case that you don't need an intermediate VM to create Docker container you should add the
line ```docker.force_host_vm = false```. Now we only need to define the container configuration for the ```nginx``` and
 ```phpfpm``` images

```ruby
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.define "phpfpm" do |phpfpm|
        phpfpm.vm.provider "docker" do |docker|
            docker.image = "phpfpm"
            docker.name = "phpfpm"
            docker.create_args = ["--volumes-from=app"]
            docker.vagrant_vagrantfile = __FILE__
        end
    end

  config.vm.define "nginx" do |nginx|
      nginx.vm.provider "docker" do |docker|
          docker.image = "nginx"
          docker.create_args = ["--volumes-from=app"]
          docker.link "phpfpm:phpfpm"
          docker.vagrant_vagrantfile = __FILE__
          docker.ports = ["80:80"]
      end
  end
end
```

Note that with this two containers we are specifying the images ```phpfpm``` and ```nginx```, that correspond to the
image names we used in the ```default``` machine into the ```docker``` provisioner. This is the full Vagrantfile

```ruby
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.define "default", autostart: false do |default|
        default.vm.box = "yungsang/boot2docker"

        # The application code
        default.vm.synced_folder ".", "/var/www", type: "nfs"

        default.vm.provider "virtualbox" do |virtualbox|
            virtualbox.memory = 2048
        end

        default.vm.provision "docker" do |docker|
            docker.build_image "/var/www/config/environment/docker/nginx", args: '-t "nginx"'
            docker.build_image "/var/www/config/environment/docker/php-fpm", args: '-t "phpfpm"'
        end
    end

    config.vm.define "app" do |app|
        app.vm.provider "docker" do |docker|
            docker.image = "debian"
            docker.name = "app"
            docker.volumes = ["/var/www:/var/www"]
            docker.vagrant_vagrantfile = __FILE__
            docker.remains_running = false
        end
    end

    config.vm.define "phpfpm" do |phpfpm|
        phpfpm.vm.provider "docker" do |docker|
            docker.image = "phpfpm"
            docker.name = "phpfpm"
            docker.create_args = ["--volumes-from=app"]
            docker.vagrant_vagrantfile = __FILE__
            docker.remains_running = true
        end
    end

  config.vm.define "nginx" do |nginx|
      nginx.vm.provider "docker" do |docker|
          docker.image = "nginx"
          docker.create_args = ["--volumes-from=app"]
          docker.link "phpfpm:phpfpm"
          docker.vagrant_vagrantfile = __FILE__
          docker.remains_running = true
          docker.ports = ["80:80"]
      end
  end
end
```

With this we are ready to run the full development environment with Docker and through Vagrant. It's important that the
containers should be created in the order specified: first the ```app``` that will hold the application source code,
next the ```phpfpm``` container that will be used by the ```nginx``` container to provide the application entry point
through the port 80. To tell Vagrant that it should create the containers in that order, we should use the
 ```--no-parallel``` flag

```bash
$ vagrant up --no-parallel
```

## Summing up

I think Docker is an awesome tool. It's pretty easy to build lightweight development environments either with Vagrant or
fig. In the worst case, we only need to spun up a tiny virtual machine before. Compared in the case where we have *n*
virtual machines one for each service or one for each application that agglutinates all the services that the application
needs. With Docker development environments will bootstrap a lot faster once they are created.

To conclude with the entry I have created a Github repo with an example application that runs its development
environment with Vagrant and Docker. Check it out at [https://github.com/theUniC/vagrant-docker-example](https://github.com/theUniC/vagrant-docker-example)

