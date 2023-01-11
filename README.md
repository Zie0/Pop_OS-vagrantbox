# Summary
This repo is to track research into vagrant boxes for [`Pop!_OS`](https://pop.system76.com/) for integration testing. Currently, this `REAMDE.md` is made of up loose notes regarding the process, roadblocks, and opportunities for improvement.

Ideally, we would utilize [`packer`](https://www.packer.io/) to build a pipeline that ingests `Pop!_OS` iso image files published by [System76](https://pop.system76.com/) and returns `Pop!_OS` vagrantboxes .

Unfortunately there are some roadblocks. The following is a non-exhuastive list of what stands between us and a vagrantbox pipeline.

## Known Knowns:
- We can trick vagrant into supporting Pop without a plugin for Pop. 
- We can create a vagrant box
  for Pop that can be interacted with via virtualbox GUI or `vagrant ssh` cli.

## Known Unknowns:
- Packer
  - Can `packer` perform an unattended install of `Pop!_OS`?
    - It can perform an unattended install of `debian` and `ubuntu`. `Pop!_OS` is downstream of `ubuntu`. However, `Pop!_OS` uses a flavor of the `ElementaryOS` installer (perhaps only the frontend of the elementary installer)
      - `ElementaryOS` does [not seem to support unattended installs](https://github.com/elementary/installer/issues/503) with [preseed files](https://wiki.debian.org/DebianInstaller/Preseed) the way Ubuntu does. However `debconf` seems to be a backend that may bypass the GTK installer
      - `Pop!_OS` uses a rust backend called [`distinst`](https://github.com/pop-os/distinst). If I understand correctly, we may be able to [target this backend with parameters that emulate selections from a UI](https://github.com/pop-os/distinst/blob/d343ec3444097afb76ebf9339a1cc736cafab1b2/tests/install.sh#L28). Gut says it would be a long shot to trick `packer` into this even if we got it working manually.
        - What do the options reference?
        - What params does the [installer](https://github.com/pop-os/installer) pass and where are they stored for the backend to read?
        - distinst used to have a config script 
      - Does `Pop!_OS` retain any of the debian lineage that may allow us to use preseed files?
      https://github.com/pop-os/distinst/pull/132
  - Does `packer` generate an [`OVF`](https://en.wikipedia.org/wiki/Open_Virtualization_Format) file when given an iso?
  - Does the `vagrant` post-processor for `packer` call `vagrant` directly on an instantiated OVF?
    - If so, working through the `vagrant` blockers may be necessary if we get that far.
- Vagrant
  - No supported [plugin for `Pop!_OS`](https://github.com/hashicorp/vagrant/tree/main/plugins/guests)
    - For the moment we can get around this by tricky Vagrant into thinking Pop is Ubuntu


# How to Make a Vagrant Base Box from an ISO (the tedious, unscalable way)
Creating a `Pop!_OS` vagrantbox from an iso does not work with Packer. We need to use Virtualbox. 
These steps are based on the [general steps for creating a base box](https://www.vagrantup.com/docs/boxes/base) and the [instructions for making one with Virtualbox](https://www.vagrantup.com/docs/providers/virtualbox/boxes)

1. [Download the `Pop!_OS`](https://pop.system76.com/) iso image

 	i. `pop-os_22.04_amd64_intel_20.iso`

2. Create a VM from the ISO using Virtualbox

   i.   Name: `pop-os_22.04_amd64_intel_20`

 	 ii.  Type: `Linux`
	
   iii. Version: `Other Linux (64-bit)`
	
   iv.  Memory Size: 4096 MB
	- System76 Recommends 4 GB RAM, 16 GB storage, 64-bit processor
	
   v.   Hard Disk: Create a virtual hard disk. Use dynamically allocated 80 GB vdi. 
	
   vi.  Do not disable peripherals (audio, USB, etc) as vagrantup.com recommends for base boxes. 
	- this is not necessary for version 1.0.0... in the future we may disable these as an optimization.

3. Install `Pop!_OS` on the VM
	
   i.   Start VM using selected ISO
	
   ii.  Select `settings > displays > resolution > 1920 x 1080`
	
   iii. Select a Language: Click "Select" on bottom right
	- "English" is highlighted by default
	
   iv.  Select a Language: Click "Select" on bottom right
	- "United States" is highlighted by default
	
   v.  Keyboard Layout: Click "Select" on bottom right
	- "English (US)..." is highlighted by default
	
   vi. Keyboard Layout; Input Language: Click "Select" on bottom right
	- "Default" is highlighted by default. 
	
   vii.  Install: Click "Clean Install"
	- Nothing is highlighted by default
	
   viii. Install: Click "Clean Install" on bottom right
	
   ix. Select a drive: Click "ATA VBOX HARDDISK"
	- Nothing is highlighted by default
	
   x.  Select a drive: Click "Erase and Install" on bottom right
	
   xi.   Create User Account:  Type "vagrant" into Full Name field
	- User Name field autopopulates to match
	
   xii.  Create User Account: Click "Next" on bottom right
	
   xiii. Create User Account; Choose Account Password: Type "vagrant" into password field 
	
   xiv.  Create User Account; Confirm Password: Type "vagrant" into password field 
	
   xv.   Create User Account; Confirm Password: Click "Next" on bottom right
	
   xvi. Drive Encryption: Click "Don't Encrypt" 
	
   xvii.  _wait while installing OS_
	
   xviii. Continue Setting Up: Click "Shut Down"
    - "Restart Device" is the other option (potentially for use with packer)
	
   xix. Virtualbox: In Virtualbox menu. uncheck the CDROM in the boot order list
	
   xx.  Virtualbox: Start VM
	
   xxi.  Login Screen: Select "vagrant"
	
   xxii. Login Screen; Password: Type "vagrant", Press `enter`
	
	--the folling steps may possibily be optional--

	 xxiii. Welcome to `Pop!_OS`; Dock options: Click "Next"
	
   xxiv. Configure the Top Bar: Click "Next"
	
   xxv. Open and Switch Applications from Launcher: Click "Next"
	
   xxvi. Use Gestures for Easier Navigation: Click "Next"
	
   xxvii. Apperance: Click "Next"
	
   xxviii. Privacy: Click "Next"
	
   xxix. Timezone:Type `Denver, Colorado, United States`
	
   xxx.  Timezone: Click "Next"
	
   xxxi. Connect Your Online Accounts: Click "Skip"
	
   xxxii. Setup Complete: Click "Start Using `Pop!_OS`"
	
   --the above steps may possibily be optional--
	
	 xxxiii. select `settings > displays > resolution > 1920 x 1080`
	
as vagrant user...

4. Passwordless sudo for vagrant user

 	i. `sudo vi /etc/sudoers` to addpend this to the file `vagrant ALL=(ALL) NOPASSWD: ALL`
	
5. install ssh server

 	i. `sudo apt update && sudo apt install openssh-server -y`
	
6. Disable SSHD UseDNS

 	i. `sudo vi /etc/ssh/sshd_config` to uncomment `UseDNS no`
	NOTE: this step is skipped in latest attempt. `UseDNS` does not appear by default in this file. It is unclear if this step remains necessary
	
7. Add `vagrant` ssh user

 	i. `mkdir ~/.ssh`

 	ii. `curl https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub > ~/.ssh/authorized_keys`

 	iii. `chmod 700 ~/.ssh`

 	iv. `chmod 600 ~/.ssh/authorized_keys`

8. Install VirtualBox Guest Additions
- Install linux kernel headers and the basic developer tools

 	i. `sudo apt-get install linux-headers-$(uname -r) build-essential dkms`
	ii. 
	> make sure that the guest additions image is available by using the [Virtualbox] GUI and clicking on "Devices" followed by "Insert Guest Additions CD Image" then `sudo sh /media/VBoxGuestAdditions/VBoxLinuxAdditions.run`
	
if the GUI doesnt work turn the below into a script that uses the same version of VBox that the VM is running in. Run the script inside the VM

```
wget https://download.virtualbox.org/virtualbox/6.1.36/VBoxGuestAdditions_6.1.36.iso
sudo mkdir /media/VBoxGuestAdditions
sudo mount -o loop,ro VBoxGuestAdditions_6.1.36.iso /media/VBoxGuestAdditions
sudo sh /media/VBoxGuestAdditions/VBoxLinuxAdditions.run
rm VBoxGuestAdditions_6.1.36.iso
sudo umount /media/VBoxGuestAdditions
sudo rmdir /media/VBoxGuestAdditions
```
	
9. spoof to Ubuntu

   i. in `/etc/os-release` change `ID` from `ID=pop` to `ID=ubuntu`
	
10. empty ~/.bash_history
	
   i. `history -c`
	
11. shutdown the VM
12. package the vagrant box

 	i. change dir to where box will be written to, `cd <foobar>`

 	ii. `vagrant package --base pop-os_22.04_amd64_intel_20 --output pop-os_22.04_amd64_intel_20.box`
  - where `--base` is the name of the VM as it is in Virtualbox and `--output` is what we want the vagrant base box to be called
	
13. Create `metadata.json` description

 	i. get checksum `sha256sum pop-os_22.04_amd64_intel_20.box`

 	ii. add it to `metadata.json` in dir with the `*.box` file
```
$ cat metadata.json
{
    "name": "ethan/pop-os_2204_amd64_intel_20",
    "short_description": "ethan's base pop-os22.04 box",
    "description": "ethan's base Pop!_OS 22.04 LTS box created from official iso using virtualbox. defaults to 4GB mem.",
    "versions": [
      {
        "version": "0.0.1",
        "providers": [
          {
            "name": "virtualbox",
            "url": "file:////home/ethan/vagrant_boxes/pop-os_22.04_amd64_intel_20.box",
            "checksum": "<checksum goes here/>",
            "checksum_type": "sha256"
          }
        ]
      }
    ]
}
```

14. Add vagrant box

 	i. `vagrant box add metadata.json`

15. ensure `vagrant-vbguest` plugin is installed
```
if [ $(vagrant plugin list | grep -c vagrant-vbguest) -eq 1 ]; then 
echo "you got it";else vagrant plugin install vagrant-vbguest; 
fi
```

16. run vagrant box

 	i. change to a new project directory

 	ii. `mkdir my-new-project; cd my-new-project`

 	iii. `vagrant init ethan/pop-os_2204_amd64_intel_20`

 	iv. `vagrant up`

17. Connect to vagrant box

 	i. `vagrant ssh`

# Packer Test Ubuntu 16

## Manual Test

In order to better understand [this boot_command](https://github.com/cbednarski/packer-ubuntu/blob/master/1604-min.json#L77) used in Hashicorp's [featured Ubuntu LTS Virtual Machines for Vagrant](https://www.packer.io/community-tools#templates) template we will manually download [this iso](https://github.com/cbednarski/packer-ubuntu/blob/master/1604-min.json#L13) and use it to build a vagrantbox manually with Virtual Box to emulate [this builder](https://github.com/cbednarski/packer-ubuntu/blob/master/1604-min.json#L74).

The idea is to the step where `boot_command` is needed and get hands on experience understanding what [Packer's Boot Configuration "special keys"](https://developer.hashicorp.com/packer/plugins/builders/virtualbox/iso#boot-configuration) actually do, how they behave, and how to use them.

![Screen Shown on Boot](https://gitlab.com/pop_os-dev/pop_os-vagrantbox/uploads/1182e4efe054bfadb210d0fa919d955f/0_boot.png "Screen Shown on Boot")*Screen Shown on Boot*

![Pressed <enter> to select "English"](https://gitlab.com/pop_os-dev/pop_os-vagrantbox/uploads/aa854a58a9bfb6eda03513aa5016ea04/1_pressed-enter.png "Pressed <enter> to select \"English\"")*(boot_command: `<enter>`) Pressed `Enter` to select "English"*

![Pressed F6 and a dialog box and Boot Options Path appear](https://gitlab.com/pop_os-dev/pop_os-vagrantbox/uploads/8bdde3888647e3c8eb1e5af8e4e41f5a/2_pressed-f6.png "Pressed F6 and a dialog box and Boot Options Path appear")*(boot_command: `<f6>`) Pressed `F6` and a dialog box and Boot Options Path appear*

![Pressed esc and dialog box is closed leaving a cursor on the Boot Options path](https://gitlab.com/pop_os-dev/pop_os-vagrantbox/uploads/552997baf90222907a3b9af277b6012c/3_pressed-esc.png "Pressed esc and dialog box is closed leaving a cursor on the Boot Options path")*(boot_command:  `<esc>`) Pressed `esc` and dialog box is closed leaving a cursor on the Boot Options path*

[![Pressed backspace 85 times. Cursor remains on a now blank boot options line](https://gitlab.com/pop_os-dev/pop_os-vagrantbox/uploads/552997baf90222907a3b9af277b6012c/3_pressed-esc.png)](https://gitlab.com/pop_os-dev/pop_os-vagrantbox/uploads/865b34429b00ba2d71c3b05cc757d309/4_pressed-bs-85-times.webm)
^^ click this. It's a video
<p class=note><em>(boot_command: <code>&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;&lt;bs&gt;</code>) Pressed <code>backspace</code> 85 times. Cursor remains on a now blank boot options line</p>


From here, we can determine what [this line](https://github.com/cbednarski/packer-ubuntu/blob/master/1604-min.json#L78) means and correlate the manual actions, if possible, to [the preseed.cfg file](https://github.com/cbednarski/packer-ubuntu/blob/master/http/preseed.cfg) it references.

https://github.com/cbednarski/packer-ubuntu/blob/master/1604-min.json

TAKE NOTE OF THIS for this readme
https://about.gitlab.com/handbook/markdown-guide/#display-local-videos-html5


# How to Make a Vagrant Base Box from an ISO (the easy way)

use Packer and preseed files?

## Setup Packer



read THIS https://developer.hashicorp.com/packer/guides/automatic-operating-system-installs/preseed_ubuntu

https://wiki.debian.org/DebianInstaller/Preseed

https://wiki.debian.org/debconf

http://www.fifi.org/doc/debconf-doc/tutorial.html

this too
https://developer.hashicorp.com/packer/plugins/builders/virtualbox/iso#optional-2

and this
https://developer.hashicorp.com/packer/plugins/builders/virtualbox/iso#boot-configuration

also this
https://github.com/cbednarski/packer-ubuntu




To create a vagrantbox using Packer with a `Pop!_OS` iso the following issues need to be resolved:

tl;dr
- preseed files didn't work on first attempt
	- likely due to Pop installer being based on elementary installer
		- Pop does an OEM install (user created after first restart). Debian does during install
- Packer config file has an "os-type" variable, and `Pop!_OS` is not an available type
	- more details here please ^^^^^
	
Preseed files didnt seem to work when we tried with `Pop!_OS`. This is likely becuase the Pop installer is derivative of the elementary installer. We may be able to work around this. Let's build an Ubuntu vagrantbox with preseed files to ensure we understand the concept.

Packer didn't seem to have elementary in it's list of OS-types. What does that var dictate? `Pop!_OS` is derivate of Ubuntu. Will this work with the correct version of Ubuntu selected?

## Building Ubuntu Vagrantbox with Packer
Here we will figure out what the preseed files need to be to build a vagrantbox with Packer.

## Packer OS-Types
Here we will determine what this var does and if it needs to change from Ubuntu to Pop in order to make Pop vagrantboxes with Packer.

# Native Vagrant Support for `Pop!_OS`
Appendix 2 shows in detail why we tricked vagrant into thinking `Pop!_OS` is `Ubuntu` in order to build a vagrant box for `Pop!_OS`. In this section we condense those lessons learned and determine how to build native or plugin support for `Pop!_OS`

## Attempt to Make Box for `Pop!_OS`
...without making Pop pretend to be ubuntu...

make a vagrant box from a VM example:
    1. `vagrant package --base pop-os_20.04-amd64 --output pop-os2004_0.1.2.box`
		2. `vagrant box add metadata.json`

when using ethan/pop-os2004
based on `galago:/mnt/sda1/vagrant/boxes/metadata.json`

```
ethan@galago:~/Projects/box_pop-os$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ethan/pop-os2004'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ethan/pop-os2004' version '0.1.2' is up to date...
==> default: Setting the name of the VM: box_pop-os_default_1605457686792_21464
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
==> default: Forwarding ports...
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Running 'pre-boot' VM customizations...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2222
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: 
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
==> default: Mounting shared folders...
    default: /vagrant => /home/ethan/Projects/box_pop-os
The guest operating system of the machine could not be detected!
Vagrant requires this knowledge to perform specific tasks such
as mounting shared folders and configuring networks. Please add
the ability to detect this guest operating system to Vagrant
by creating a plugin or reporting a bug.
```

Vagrant is unable to detect the operating system (`Pop!_OS`) of the virtual machine.
    https://www.vagrantup.com/docs/vagrantfile/machine_settings.html#config-vm-guest
    > This defaults to :linux, and Vagrant will auto-detect the proper distro.

It appears vagrant is not auto-detecting `Pop!_OS`. How do we get vagrant to detect `Pop!_OS`?

A plugin for `Pop!_OS` may need to be written for vagrant. This example shows how the plugin for `Ubuntu` was implemented.
    https://github.com/hashicorp/vagrant/tree/main/plugins/guests/ubuntu

Since `Pop!_OS` is downstream of `Ubuntu` let's model the `Pop!_OS` plugin after `Mint`
    https://github.com/hashicorp/vagrant/tree/main/plugins/guests/mint
        ~> note: this dir structure is identical to the `../ubuntu/` dir. It contains `guest.rb` and `plugin.rb`. It does not contain `cap/` like `../debian.`

plugins/guests/linux/guest.rb:
GUEST_DETECTION_NAME

## Requirements for `Pop!_OS` Plugin
- non-exhaustive, needs scoping

plugin dir located in `vagrant/plugins/guests/pop/`

contains `guest.rb` and `plugin.rb`

`guest.rb` like <code block\> where `GUEST_DETECTION_NAME` = `"Pop"` or `pop`?

```ruby
require_relative '../linux/guest'

module VagrantPlugins
  module GuestMint
    class Guest < VagrantPlugins::GuestLinux::Guest
      # Name used for guest detection
      GUEST_DETECTION_NAME = "Linux Mint".freeze
    end
  end
end																			 
```

# Appendix (Resources for `Pop!_OS` Vagrant Plugin)

https://www.vagrantup.com/docs/plugins/guests.html
> Vagrant has many features that requires doing guest OS-specific actions, such as mounting folders, configuring networks, etc. These tasks vary from operating system to operating system. If you find that one of these does not work for your operating system, then maybe the guest implementation is incomplete or incorrect.
    https://www.vagrantup.com/docs/plugins/development-basics
https://www.vagrantup.com/docs/plugins/guest-capabilities


# Appendix 2 - Ruby dev env for vagrant plugin config
References 
	[Setup a Ruby Dev Env](https://www.jetbrains.com/help/ruby/set-up-a-ruby-development-environment.html)

Getting setup
	
[Install Ruby with a snap package](https://www.ruby-lang.org/en/documentation/installation/#snap)
`sudo snap install ruby --classic`
	
The above installation method includes Bundler ( usage: `bundle -h` ) and RubyGems ( usage: `gem -h` )
	

	
# Appendix 3 (Spoofing `/etc/os-release`)

## Abridged

1. In `Pop!_OS` VirtualBox VM

Changed `/etc/os-release` values
ln 1:`Name="Ubuntu"`
ln 3:`ID=ubuntu`

2. next, build the VM into a vagrant box:

`vagrant package --base pop-os_20.04-amd64 --output pop-os2004-ubuntuhack_0.1.0.box` 

3. add Vagrantbox to vagrant.d/
create `metadata.json`
```
{
    "name": "ethan/pop-os2004-ubuntuhack",
    "short_description": "ethan's base pop-os20.04 box pretending to be ubuntu",
    "description": "ethan's base Pop!_OS 20.04 LTS box created from official (non-nvidia) iso using virtualbox. defaults to 2GB mem. changed /etc/os-release to pretend to be ubuntu in effort to trick vagrant",
    "versions": [
      {
        "version": "0.1.0",
        "providers": [
          {
            "name": "virtualbox",
            "url": "file:///mnt/sda1/vagrant/boxes/pop-os2004-ubuntuhack/pop-os2004-ubuntuhack_0.1.0.box",
            "checksum": "6717ae04b0577fca29159456349918f63bcf67ffe8b8beae246606547e2663ce",
            "checksum_type": "sha256"
          }
        ]
      }
    ]
  }  
```

`vagrant box add metadata.json`

5. ensure `vagrant-vbguest` plugin is installed
`vagrant plugin install vagrant-vbguest`

6. start vagrant box

`vagrant up`

### Outstanding issues
0. vagrant is sync'ing the default dir _and_ the dir specified. we want just the spec'd dir sync'd
    https://stackoverflow.com/questions/18528717/vagrant-shared-and-synced-folders
1. Virtualbox Guest Additions mismatch and error on install
Vagrant says 
> [default] A Virtualbox Guest Additions installation was found but no tools to rebuild or start them.

So it copies VBoxGuestAdditions into the box
> Copy iso file /home/ethan/.config/VirtualBox/VBoxGuestAdditions_6.1.14.iso into the box /tmp/VBoxGuestAdditions.iso

and installs version 6.1.14 onto the box
> Installing Virtualbox Guest Additions 6.1.14 - guest version is 6.1.6

during install, Vagrant says the box will need a restart
> VirtualBox Guest Additions: Running kernel modules will not be replaced until the system is restarted

Vagrant says an error occurs during installation of `VirtualBox Guest Additions`
> An error occurred during installation of VirtualBox Guest Additions 6.1.14. Some functionality may not work as intended.
In most cases it is OK that the "Window System drivers" installation failed.
Unmounting Virtualbox Guest Additions ISO from: /mnt
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...

## Detail
Hacked a `Pop!_OS` VM to pretend it's ubuntu for vagrant. 

attempting to leverage these files:
    https://github.com/hashicorp/vagrant/blob/main/plugins/guests/ubuntu/guest.rb
    https://github.com/hashicorp/vagrant/blob/main/plugins/guests/linux/guest.rb

Changed `/etc/os-release` values
ln 1:`Name="Ubuntu"`
ln 3:`ID=ubuntu`

next, build a vagrant box:

	`vagrant package --base pop-os_20.04-amd64 --output pop-os2004-ubuntuhack_0.1.0.box` 

create `metadata.json`
```
{
    "name": "ethan/pop-os2004-ubuntuhack",
    "short_description": "ethan's base pop-os20.04 box pretending to be ubuntu",
    "description": "ethan's base Pop!_OS 20.04 LTS box created from official (non-nvidia) iso using virtualbox. defaults to 2GB mem. changed /etc/os-release to pretend to be ubuntu in effort to trick vagrant",
    "versions": [
      {
        "version": "0.1.0",
        "providers": [
          {
            "name": "virtualbox",
            "url": "file:///mnt/sda1/vagrant/boxes/pop-os2004-ubuntuhack/pop-os2004-ubuntuhack_0.1.0.box",
            "checksum": "6717ae04b0577fca29159456349918f63bcf67ffe8b8beae246606547e2663ce",
            "checksum_type": "sha256"
          }
        ]
      }
    ]
  }  
```

`vagrant box add metadata.json`
`vagrant up`

```
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
==> default: Mounting shared folders...
    default: /vagrant => /home/ethan/Projects/box_pop-os
Vagrant was unable to mount VirtualBox shared folders. This is usually
because the filesystem "vboxsf" is not available. This filesystem is
made available via the VirtualBox Guest Additions and kernel module.
Please verify that these guest additions are properly installed in the
guest. This is not a bug in Vagrant and is usually caused by a faulty
Vagrant box. For context, the command attempted was:

mount -t vboxsf -o uid=1000,gid=1000 vagrant /vagrant

The error output from the command was:

mount: /vagrant: wrong fs type, bad option, bad superblock on vagrant, missing codepage or helper program, or other error.
```




troubleshooting:
https://stackoverflow.com/questions/43492322/vagrant-was-unable-to-mount-virtualbox-shared-folders
`vagrant ssh`
```
vagrant@pop-os:~$ ls -lh /sbin/mount.vboxsf 
ls: cannot access '/sbin/mount.vboxsf': No such file or directory
vagrant@pop-os:~$ ls /opt/VBoxGuestAdditions-*/lib/VBoxGuestAdditions/mount.vboxsf
ls: cannot access '/opt/VBoxGuestAdditions-*/lib/VBoxGuestAdditions/mount.vboxsf': No such file or directory
```

ran 
`vagrant plugin install vagrant-vbguest`
`vagrant destroy`
`vagrant up`

IT WORKED!!!

```ethan@galago:~/Projects/box_pop-os$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ethan/pop-os2004-ubuntuhack'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ethan/pop-os2004-ubuntuhack' version '0.1.0' is up to date...
==> default: Setting the name of the VM: box_pop-os_default_1605473917907_37027
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
==> default: Forwarding ports...
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Running 'pre-boot' VM customizations...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2222
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: 
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default: 
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
[default] A Virtualbox Guest Additions installation was found but no tools to rebuild or start them.
Reading package lists...
Building dependency tree...
Reading state information...
dkms is already the newest version (2.8.1-5ubuntu1).
dkms set to manually installed.
linux-headers-5.4.0-7642-generic is already the newest version (5.4.0-7642.46~1598628707~20.04~040157c).
linux-headers-5.4.0-7642-generic set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 288 not upgraded.
Copy iso file /home/ethan/.config/VirtualBox/VBoxGuestAdditions_6.1.14.iso into the box /tmp/VBoxGuestAdditions.iso
Mounting Virtualbox Guest Additions ISO to: /mnt
mount: /mnt: WARNING: device write-protected, mounted read-only.
Installing Virtualbox Guest Additions 6.1.14 - guest version is 6.1.6
Verifying archive integrity... All good.
Uncompressing VirtualBox 6.1.14 Guest Additions for Linux........
VirtualBox Guest Additions installer
Copying additional installer modules ...
Installing additional modules ...
VirtualBox Guest Additions: Starting.
VirtualBox Guest Additions: Building the VirtualBox Guest Additions kernel 
modules.  This may take a while.
VirtualBox Guest Additions: To build modules for other installed kernels, run
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup <version>
VirtualBox Guest Additions: or
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup all
VirtualBox Guest Additions: Building the modules for kernel 5.4.0-7642-generic.
update-initramfs: Generating /boot/initrd.img-5.4.0-7642-generic
cryptsetup: WARNING: Resume target cryptswap uses a key file
VirtualBox Guest Additions: Running kernel modules will not be replaced until 
the system is restarted
An error occurred during installation of VirtualBox Guest Additions 6.1.14. Some functionality may not work as intended.
In most cases it is OK that the "Window System drivers" installation failed.
Unmounting Virtualbox Guest Additions ISO from: /mnt
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...
Restarting VM to apply changes...
==> default: Attempting graceful shutdown of VM...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
==> default: Mounting shared folders...
    default: /vagrant => /home/ethan/Projects/box_pop-os
    default: /vagrant_data => /home/ethan/Projects/box_pop-os/src
```

Shared folders work!!
    it isn't the cleanest right now because we have two shared directories.
        the default
            /vagrant => /home/ethan/Projects/box_pop-os
        and the Vagrantfile defined
            default: /vagrant_data => /home/ethan/Projects/box_pop-os/src

was this resolved simply by running this earlier?
    `vagrant plugin install vagrant-vbguest`
or was it because we spoofed `/etc/os-release` in `ethan/pop-os2004-ubuntuhack`?

trying this with `ethan/pop-os2004`

we found that vagrant requires the spoofed `/etc/os-release` && `vagrant plugin install vagrant-vbguest` having been run before `vagrant up` in order for shared_folders to work

results with `ethan/pop-os2004` && no spoofed `/etc/os-release`
```
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
The guest operating system of the machine could not be detected!
Vagrant requires this knowledge to perform specific tasks such
as mounting shared folders and configuring networks. Please add
the ability to detect this guest operating system to Vagrant
by creating a plugin or reporting a bug.
```

returning to using `ethan/pop-os2004-ubuntuhack`...
we want to test reproduceability:
`vagrant destroy`
`vagrant up`

```
ethan@galago:~/Projects/box_pop-os$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ethan/pop-os2004-ubuntuhack'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ethan/pop-os2004-ubuntuhack' version '0.1.0' is up to date...
==> default: Setting the name of the VM: box_pop-os_default_1605474998454_84525
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
    default: Adapter 1: nat
==> default: Forwarding ports...
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Running 'pre-boot' VM customizations...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2222
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: 
    default: Vagrant insecure key detected. Vagrant will automatically replace
    default: this with a newly generated keypair for better security.
    default: 
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
[default] A Virtualbox Guest Additions installation was found but no tools to rebuild or start them.
Reading package lists...
Building dependency tree...
Reading state information...
dkms is already the newest version (2.8.1-5ubuntu1).
dkms set to manually installed.
linux-headers-5.4.0-7642-generic is already the newest version (5.4.0-7642.46~1598628707~20.04~040157c).
linux-headers-5.4.0-7642-generic set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 288 not upgraded.
Copy iso file /home/ethan/.config/VirtualBox/VBoxGuestAdditions_6.1.14.iso into the box /tmp/VBoxGuestAdditions.iso
Mounting Virtualbox Guest Additions ISO to: /mnt
mount: /mnt: WARNING: device write-protected, mounted read-only.
Installing Virtualbox Guest Additions 6.1.14 - guest version is 6.1.6
Verifying archive integrity... All good.
Uncompressing VirtualBox 6.1.14 Guest Additions for Linux........
VirtualBox Guest Additions installer
Copying additional installer modules ...
Installing additional modules ...
VirtualBox Guest Additions: Starting.
VirtualBox Guest Additions: Building the VirtualBox Guest Additions kernel 
modules.  This may take a while.
VirtualBox Guest Additions: To build modules for other installed kernels, run
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup <version>
VirtualBox Guest Additions: or
VirtualBox Guest Additions:   /sbin/rcvboxadd quicksetup all
VirtualBox Guest Additions: Building the modules for kernel 5.4.0-7642-generic.
update-initramfs: Generating /boot/initrd.img-5.4.0-7642-generic
cryptsetup: WARNING: Resume target cryptswap uses a key file
VirtualBox Guest Additions: Running kernel modules will not be replaced until 
the system is restarted
An error occurred during installation of VirtualBox Guest Additions 6.1.14. Some functionality may not work as intended.
In most cases it is OK that the "Window System drivers" installation failed.
Unmounting Virtualbox Guest Additions ISO from: /mnt
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...
Restarting VM to apply changes...
==> default: Attempting graceful shutdown of VM...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
==> default: Mounting shared folders...
    default: /vagrant => /home/ethan/Projects/box_pop-os
    default: /vagrant_data => /home/ethan/Projects/box_pop-os/src
ethan@galago:~/Projects/box_pop-os$ 
```

```
$ vboxmanage --version
6.1.14_Popr140239

$ vagrant --version
Vagrant 2.2.9

$ vagrant plugin list
vagrant-vbguest (0.27.0, global)
```

Vagrant says 
> [default] A Virtualbox Guest Additions installation was found but no tools to rebuild or start them.

So it copies VBoxGuestAdditions into the box
> Copy iso file /home/ethan/.config/VirtualBox/VBoxGuestAdditions_6.1.14.iso into the box /tmp/VBoxGuestAdditions.iso

and installs version 6.1.14 onto the box
> Installing Virtualbox Guest Additions 6.1.14 - guest version is 6.1.6

during install, Vagrant says the box will need a restart
> VirtualBox Guest Additions: Running kernel modules will not be replaced until the system is restarted

Vagrant says an error occurs during installation of `VirtualBox Guest Additions`
> An error occurred during installation of VirtualBox Guest Additions 6.1.14. Some functionality may not work as intended.
In most cases it is OK that the "Window System drivers" installation failed.
Unmounting Virtualbox Guest Additions ISO from: /mnt
Got different reports about installed GuestAdditions version:
Virtualbox on your host claims:   6.1.6
VBoxService inside the vm claims: 6.1.14
Going on, assuming VBoxService is correct...

VirtualBox on host is version 6.1.14 but vagrant throwing this message `Virtualbox on your host claims:   6.1.6`




to try:
re-installing VBoxGuestAdditions on the VM and making it into a new vagrant box

https://stackoverflow.com/questions/28328775/virtualbox-mount-vboxsf-mounting-failed-with-the-error-no-such-device/29456128#29456128

https://stackoverflow.com/questions/43733108/vagrant-up-got-different-reports-about-installed-guestadditions-version

https://stackoverflow.com/a/26902612 ?
	
	
	# Appendix Random Research
	creating a vagrant box from within a VM
https://www.vagrantup.com/docs/providers/virtualbox/boxes.html

what is ovf?
https://en.wikipedia.org/wiki/Open_Virtualization_Format

what is packer?
>HashiCorp Packer automates the creation of any type of machine image. It embraces modern configuration management by encouraging you to use automated scripts to install and configure the software within your Packer-made images

https://www.packer.io/docs/terminology

what is a packer builder?
https://www.packer.io/docs/builders
https://www.packer.io/docs/builders/virtualbox-iso

perspective on how to build a vagrant box from scratch and what packer automates
https://blog.engineyard.com/building-a-vagrant-box

packer build an image
https://learn.hashicorp.com/tutorials/packer/getting-started-build-image?in=packer/getting-started

why a vagrantbox and not just a vm?
https://www.quora.com/Why-should-I-use-Vagrant-instead-of-creating-multiple-VMs-in-VirtualBox-or-VMWare-Workstation?share=1
pretty much.. bc vagrant shared directories and port mapping type features allow changes to be made to the guest system while using tools in host system (less to provision to become productive!)
https://www.codehenge.net/2013/02/automate-your-development-environment-with-vagrant/
> But wait, we've been able to create virtual machines for years. What's new here? The problem is configuration of a brand new virtual macine for each project is a massive chore, reinstalling all of your dev tools each time sounds like torture, and developers will still each want VMs of different operating systems...what are we solving? Vagrant does it differently. By giving you the option to leverage powerful, proven automated configuration technologies such as Chef or Puppet (as well as your own custom shell scripts, if you like), Vagrant takes the time and tedium out of configuring a virtual environment.

more why vagrant
https://www.codehenge.net/2013/02/automate-your-development-environment-with-vagrant/

GUI on vagrant managed virtualbox box 
https://stackoverflow.com/questions/20227140/can-i-bring-up-the-gui-for-a-vagrant-managed-virtual-box-while-the-box-is-runnin

creating a vagrant box from an existing vagrant box that has been provisioned
https://www.packer.io/docs/builders/vagrant

ancient how-to build vagrantbox  from iso thread
https://stackoverflow.com/questions/15243405/is-it-possible-for-vagrant-to-use-an-os-iso-install-image-directly-or-create-a