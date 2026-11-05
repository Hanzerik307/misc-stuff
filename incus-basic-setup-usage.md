# Setting Up Incus: Server and Desktop Client

## The Starting Point

In this example I have a bare minimal Debian "Server" that I will use to install the Incus suite on. It is a VM running via Incus on my main Debian + Incus server. The server VM is set up with 6 CPU cores, 12 GB RAM, a 30 GB disk image, and an ext4 filesystem.

The basics are already done:

- My user has been added to sudo on the server.
- The SSH server is running on the server.
- Core basic system utilities are installed on the server.
- I copied my SSH key from the desktop to the server VM so I don't have to type a password every time.

I should also mention the server is connected to the network via a `br0` device on my main server, so it can talk to other PCs on the LAN, just like a real bare-metal setup.

## Installing Incus on the Server

So we have a very lean server running nothing but `sshd`. At this point I install the Incus suite from <https://github.com/zabbly/incus>.

Incus LTS is available through the normal Debian repos, but what I like about installing the Stable version from the Incus GitHub is that we get newer packages and features — plus we can install the Web UI from there too. I won't go into the details of that, because it's up to you whether you install from the Zabbly repo or from the Debian repos. Everything I'm going to do later can be done with either version.

To install Incus on the server using either method (Debian repo or Zabbly repo):

```bash
sudo apt update && sudo apt upgrade
sudo apt install incus
```

Doing this will install all of the required dependencies and Incus components. Installing from Zabbly will give you different versions of some packages, specifically built for the Zabbly Incus packages. If you want the Web UI from Zabbly, you can install it with:

```bash
sudo apt install incus-ui-canonical
```

If you go the Debian route, it uses the standard QEMU/KVM packages available from Debian and the standard LTS Incus packages — the Web UI is not available that way.

If you don't want to run VMs and only want to run containers, you can install `incus-base` instead of `incus`. But since we are talking about running VMs on Incus, we'll install the whole `incus` suite.

## Initializing Incus

Once it's installed, we'll add our user to the `incus-admin` group:

```bash
sudo usermod -aG incus-admin <username>
```

Log out and log back in again, then run:

```bash
incus admin init
```

No `sudo` is needed here, because we added our user to the `incus-admin` group.

It will ask a bunch of questions, and you should be able to accept the defaults. The one you will change is the question about being accessible over the network. This way you can access and view your VM desktops and consoles from another machine on your network.

```
hanzerik@incus-server:~$ incus admin init
Would you like to use clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Name of the storage backend to use (dir, truenas) [default=dir]:
Where should this storage pool store its data? [default=/var/lib/incus/storage-pools/default]:
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=incusbr0]:
What IPv4 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]:
What IPv6 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]:
Would you like the server to be available over the network? (yes/no) [default=no]: yes
Address to bind to (not including port) [default=all]:
Port to bind to [default=8443]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "init" preseed to be printed? (yes/no) [default=no]: no
```

## Launching a Test VM

Incus is now set up and waiting for us to create our first instance. To play around, we'll just fire up a basic Debian VM with the defaults:

```bash
hanzerik@incus-server:~$ incus launch images:debian/13 test-vm --vm
Launching test-vm
```

That command will download a bare Debian 13 image from `linuxcontainers.org`, launch an instance called `test-vm` with the default VM settings of **1 CPU, 1 GB RAM, and a 10 GB disk**, and use the `incusbr0` bridge network device set up during `incus admin init`.

That's not a very powerful VM, but a small HTTP/Git server could be set up on it. You can choose from a lot of different images on `linuxcontainers.org` — I use Debian because that's what I know best.

Next, run:

```bash
incus list
```

The `incus list` command lists all your instances, their status, and their IP, whether they are containers or VMs.

## Working with the Test VM

To work with the new VM we have a couple of options. The first is not to enter it at all, and instead use:

```bash
incus exec test-vm -- free -h
```

That prints the output of `free -h` from inside the VM. Personally, I enter the VM using:

```bash
incus shell test-vm
```

Now I am inside the VM as `root`. At this point I add another normal sudo user using my standard Linux commands, and stop running as `root`.

OK, we don't want that VM anymore, so we'll stop and delete it:

```bash
incus stop test-vm
incus delete test-vm
```

## Setting Up the Desktop Client

Now we'll set up our desktop/laptop to manage the Incus service running on our server. This way we don't have to SSH into the server to create, modify, or delete containers and VMs.

Depending on what distro you use, you can install just the `incus-client` (Debian, Ubuntu, and Fedora have just the client app available). On other distros you can download just the client binary from the Zabbly GitHub and plop it in, say, `~/.local/bin`. So on a Debian desktop, you can just do:

```bash
sudo apt install incus-client
```

If you are wanting to connect to a VM with a desktop, go ahead and install `virt-viewer`.

I forgot to mention: you can also install Incus on your desktop/laptop and not use a remote server at all if you don't have one. I prefer using Incus over `Virt-Manager` and `VirtualBox` on my desktops/laptops if I want to run local VMs. All these steps are the same, except the part where we allowed access over the network during the Incus init, and the steps that follow.

## Establishing Trust Between Client and Server

Now we have our client app installed on our PC, and we want to set up trust between our PC and the server.

On your PC, open two terminals. In the first terminal, SSH into the server and run:

```bash
incus config trust add <your-pc-name>
```

This will print out a long trust key token — leave it open for now.

In the second terminal, on the PC, run:

```bash
incus remote add <ip-or-hostname-of-server>
```

This will show you a certificate fingerprint; answer `y` to the prompt. It will ask for the Trust Token that was printed out in the terminal on the server. Paste that token into the terminal on the PC and hit enter. It will now say `Client Certificate now trusted by server`.

You can see that your PC is now trusted by the server by running this on the server:

```bash
incus config trust list
```

It will show the PC name, fingerprint, and expiration date of the trust. You can now exit the SSH session with the server — we won't need to access it again unless we want to update or upgrade packages on the server itself.

## Switching Remotes

On the PC, do:

```bash
incus remote switch <ip-or-hostname-of-server>
```

(I can use naked hostnames on my LAN, so I never use IPs.)

Now we can run `incus list` and see that we have no instances created — unless you didn't delete the one from earlier. We can create a VM just like we did earlier if we wanted to, but what we want is a normal desktop VM up and running. We can use the images from `linuxcontainers.org` or use our favorite distro ISO. For the following examples I'll be using the Debian Netinstall image downloaded from the Debian website, but you can use whatever you want.

## Creating a Reusable VM Profile

As you saw earlier, the default profile for a VM is pretty weak. We want a VM that is usable and within the limits of our server host as far as resources go. So I like to use profiles instead of configuring each VM every time I launch one.

First, copy the default profile:

```bash
incus profile copy default vm-normal
```

I'm using `vm-normal`, but you could also go with something like `vm-small`, `vm-large`, `vm-extra-large`, etc., depending on how many resources you want to allocate to the VMs you create. Currently `vm-normal` is exactly the same as the default, but we will change that.

For a simple desktop, and to stay within the limits of my "Server VM", I'm going to set the CPU limit for `vm-normal` to 4 CPUs:

```bash
incus profile set vm-normal limits.cpu=4
```

And memory to 6 GiB:

```bash
incus profile set vm-normal limits.memory=6GiB
```

The disk space is still at the default of 10 GiB, so we want to update that to something that can handle a desktop distro. My "Server VM" is pretty small, so I'll just go to 20 GiB:

```bash
incus profile device set vm-normal root size=20GiB
```

Now, when we configure or start a normal `linuxcontainers.org` image, we can make it use this new profile to have these resources allocated to it like so:

```bash
incus launch images:debian/13 my-vm --vm --profile=vm-normal
```

## Creating a VM from an ISO

We want to use our favorite distro ISO, so we're going to upload the ISO file to the Incus server:

```bash
incus storage volume import default Downloads/debian-13.7.0-amd64-netinst.iso install-iso --type=iso
```

This uploads the ISO file from your PC to the storage pool on the server and makes it available as `install-iso` — you can name it anything you wish.

Now we are all set to make our first VM from an ISO file. We need to create an empty VM that uses our `vm-normal` profile:

```bash
incus init debian-vm --empty --vm --profile=vm-normal
```

And then attach the ISO as a CD-ROM disk drive:

```bash
incus config device add debian-vm cdrom disk pool=default source=install-iso boot.priority=10
```

As you can see, this amount of typing is why I prefer to use the Web UI from Zabbly to do this. `incus webui` fires up your default browser and connects to the Incus server. You can manage everything via a web interface, including uploading ISO files via the GUI.

## Installing the Guest OS

Our VM is now created and ready to install. Make sure at this point you have `virt-viewer` installed on your PC.

Open another terminal window, which will be used later to remove the ISO file from the VM after installation.

If you are installing an Arch Linux distro, or any distro that requires Secure Boot to be disabled, there is another step you need to take:

```bash
incus config set <vm-name> security.secureboot=false
```

Now we are ready to fire up the VM:

```bash
incus start <vm-name> --console=vga
```

This starts up `virt-viewer`, connects to the VM desktop, and you can begin the install process.

When you are done installing, some distros ask that you remove the install media and hit enter/reboot — this is where the second terminal comes into play. We're going to remove the installation media with:

```bash
incus config device remove <vm-name> cdrom
```

If you forget to remove the media and it boots back into the installation screen, you can force stop the VM with:

```bash
incus stop <vm-name> --force
```

Then run the command above to remove the ISO and start it back up.

## Managing Instances

You can manage the state of an instance without connecting to it. Say you have a container or VM set up as a web server (I use `linuxcontainers.org` images for these types of services) — you can just run:

```bash
incus start <vm-name>
incus stop <vm-name>
incus restart <vm-name>
```

For VMs installed from an ISO that don't have `incus-agent` installed, unlike the `linuxcontainers.org` images, you may have to issue the `--force` flag to stop them.

### Handy Commands

| Task | Command |
| --- | --- |
| Start a VM and connect to its desktop | `incus start <vm-name> --console=vga` |
| Connect to the console of a running VM | `incus console <vm-name> --type=vga` |
| Connect to a VM/container from a `linuxcontainers.org` image | `incus shell <name>` |


### Stuff to add for running Incus locally on Omarchy installed from EXTRA:

```bash
sudo usermod -v 1000000-1000999999 -w 1000000-1000999999 root
sudo systemctl enable incus.service
sudo systemctl enable incus.socket
```

```bash
sudo ufw allow in on incusbr0
sudo ufw route allow in on incusbr0
sudo ufw route allow out on incusbr0

sudo reboot
```

