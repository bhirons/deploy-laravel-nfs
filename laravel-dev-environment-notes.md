# Laravel development environment

These are my notes about my Laravel dev environment, and includes a few mix-ins related to developing and deploying on nearlyfreespeeck.net. This is linked from the [README document](README.md).

This is on the Mac OSX and utilizes [Homestead](https://laravel.com/docs/master/homestead).

I recomend a VM for web application development since it keeps a lot of dev environment stuff off of our host machine where you might also have other critical or personal processes. For PHP in general and Laravel in specific Homestead is the best option for me. I also use it to run local copies of wordpress sites for plugin development.

## My environment

All the code is in a single folder that is mapped into the VM, this means that edits are always in sync in the guest VM and your host local machine because it **is** the same file. 

I run my editor on my host machine. I like VSCode right now because it is usable, pretty good, easy, very configurable, and free.

I run the VM for my development web server, and I use a terminal to shell into the vm to run all project and platform specific commands. I run composer, npm as needed, gulp, and Laravel artisan commands inside the VM.

I also have a host terminal open to the project folder and I run all my git commands on my host machine. I have some git configurations that are convenient to set once and use everywhere, so they persist beyond any given VM that comes and goes.

### Homestead

You need a virtual machine, I use [VirtualBox](https://www.virtualbox.org/wiki/Downloads) because it performs well on my Mac and the price is right.

You need [vagrant](https://www.vagrantup.com/downloads.html), which is a tool for managing virtual machines.

Get the VM running and then you may [install Homestead](https://laravel.com/docs/master/homestead).

#### Homestead.yaml
Here is a basic `Homestead.yaml` file that lives in the Homestead folder created during installation:

```
---
ip: "192.168.10.10"
memory: 2048
cpus: 1
provider: virtualbox

authorize: ~/.ssh/id_rsa.pub

keys:
    - ~/.ssh/id_rsa

ssl: true

folders:
    - map: /Users/buddhironsjr/Projects
      to: /home/vagrant/code

sites:
    - map: laravelproject.local
      to: /home/vagrant/code/laravelproject/public
    - map: htmlsite.local
      to: /home/vagrant/code/htmlsite
    - map: generatedsite.local
      to: /home/vagrant/code/generatedsite/dist
   
databases:
    - homestead

features:
    - mariadb: true
```

I keep all my projects in `~/Projects`, and you can see above in the `folders` section how I map it in the vm.

Note that each site will be served on a `.local` domain, and that the sites entry makes to the appropriate folder for a laravel project.  The other sites entry is fgor a regular html project that doesn't use its own public folder.

#### Database change

nearlyfreespeech.net runs MariaDB which is binary compatible with MySQL, yet Homestead uses MySQL by default  This difference between your productiron environment and the dev environment is not healthy. And causes more work when deployng.

So make Homestead use MariaDB as shown in the `features` section above, and it will swap out the installed database provider for you. You will still configure the database using mysql driver information in any application configuration.

### Local host names

Use the IP address above to create a local hosts entrie for your projects in `/etc/hosts` on your **host** system.

    $ sudo nano /etc/hosts

Here is an example of what mine may look like

```
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
192.168.10.10   laravelproject.local
192.168.10.10   htmlsite.local
192.168.10.10   generatedsite.local
::1             localhost
```