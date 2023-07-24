---
title: Setup
---


There are two pre-requisites for this workshop:

1. Installing SingularityCE/Apptainer locally on your laptop so you can build container images - note: you must have access to a laptop where you have admin/root privileges.
2. Having an account on a remote HPC system to be able to run the Singularity+MPI containers - if you do not have a system where you can do this, you can sign up for an account on ARCHER2 as described below for this part of the workshop.

Below, we cover instructions for setting up SingularityCE/Apptainer on your laptop and how to sign up for an account on ARCHER2 if you need it.

## Install SingularityCE/Apptainer on your laptop

Note: you must have admin/root privileges on the laptop you are using to install SingularityCE/Apptainer. For Linux and Windows, we provide instructions for installing SingularityCE and for macOS, we provide instructions for installing Apptainer. SingularityCE and Apptainer are currently mutually compatible but this may change in the future.

### Linux SingularityCE installation

Follow the instructions for installing SingularityCE from the documentation:

 - [Building SingularityCE from source](https://docs.sylabs.io/guides/3.11/admin-guide/installation.html#install-from-source)

### macOS Apptainer installation

First, install the Lima virtual machine framework using Homebrew. Open a macOS Terminal and use:

```
$ brew install qemu lima
```

Next download the Apptainer Lima configuration:

```
$ limactl start template://apptainer
$ limactl stop apptainer
```

By default, the home directory in the VM will not be writeable. You should edit the `apptainer` template config to make it writeable with:

```
$ limactl edit apptainer
```

Find the `mounts` section and change the line `- location: "~"` to:

```
mounts:
- location: "~"
  writable: true
```

You can now start the Apptainer VM and get an interactive shell:

```
$ limactl start apptainer

INFO[0000] Using the existing instance "apptainer"      
INFO[0000] [hostagent] Starting QEMU (hint: to watch the boot progress, see "/Users/auser/.lima/apptainer/serial.log") 
INFO[0000] SSH Local Port: 54642                        
INFO[0000] [hostagent] Waiting for the essential requirement 1 of 5: "ssh" 
INFO[0010] [hostagent] Waiting for the essential requirement 1 of 5: "ssh" 
INFO[0011] [hostagent] The essential requirement 1 of 5 is satisfied 
INFO[0011] [hostagent] Waiting for the essential requirement 2 of 5: "user session is ready for ssh" 
INFO[0021] [hostagent] Waiting for the essential requirement 2 of 5: "user session is ready for ssh" 
INFO[0021] [hostagent] The essential requirement 2 of 5 is satisfied 
INFO[0021] [hostagent] Waiting for the essential requirement 3 of 5: "sshfs binary to be installed" 
INFO[0021] [hostagent] The essential requirement 3 of 5 is satisfied 
INFO[0021] [hostagent] Waiting for the essential requirement 4 of 5: "/etc/fuse.conf (/etc/fuse3.conf) to contain \"user_allow_other\"" 
INFO[0021] [hostagent] The essential requirement 4 of 5 is satisfied 
INFO[0021] [hostagent] Waiting for the essential requirement 5 of 5: "the guest agent to be running" 
INFO[0021] [hostagent] The essential requirement 5 of 5 is satisfied 
INFO[0021] [hostagent] Mounting "/Users/auser" on "/Users/auser" 
INFO[0022] [hostagent] Mounting "/tmp/lima" on "/tmp/lima" 
INFO[0022] [hostagent] Waiting for the optional requirement 1 of 1: "user probe 1/1" 
INFO[0022] [hostagent] Forwarding "/run/lima-guestagent.sock" (guest) to "/Users/auser/.lima/apptainer/ga.sock" (host) 
INFO[0022] [hostagent] Forwarding TCP from [::]:5355 to 127.0.0.1:5355 
INFO[0022] [hostagent] The optional requirement 1 of 1 is satisfied 
INFO[0022] [hostagent] Waiting for the final requirement 1 of 1: "boot scripts must have finished" 
INFO[0022] [hostagent] Not forwarding TCP [::]:22       
INFO[0022] [hostagent] Not forwarding TCP 127.0.0.53:53 
INFO[0022] [hostagent] Not forwarding TCP 127.0.0.54:53 
INFO[0022] [hostagent] Forwarding TCP from 0.0.0.0:5355 to 127.0.0.1:5355 
WARN[0022] [hostagent] failed to set up forwarding tcp port 5355 (negligible if already forwarded) 
INFO[0022] [hostagent] Not forwarding TCP 0.0.0.0:22    
INFO[0022] [hostagent] The final requirement 1 of 1 is satisfied 
INFO[0022] READY. Run `limactl shell apptainer` to open the shell. 

$ limactl shell apptainer
[auser@lima-apptainer ~]$ singularity --version
apptainer version 1.1.7-1.fc38
```

### Windows SingularityCE installation

This process has been tested but there are other potential approaches to installing SingularityCE (or Apptainer)
via WSL2. We have had less success with other approaches so detail the most reliable one below.

We use WSL2 to install SingularityCE on Windows 10 or 11. First, make sure that a suitable WSL install is
available (if oyu already have Ubuntu available through WSL, you can skip this step). Open Powershell on
your system and use:

```
wsl --install -d Ubuntu-20.04
```

You will be prompted for a username and password, you are free to choose these to be whatever you want.

Next, you need to install the SingularityCE prerequisites. Open Powershell and type `wsl` to get a Linux prompt.
Install the Go programming language development environment and other prerequisites:

```
sudo apt update
sudo apt full-upgrade
sudo apt install golang-go 
```

Followed by other prerequisites:

```
sudo apt-get update && sudo apt-get install -y \
    build-essential \
    uuid-dev \
    libgpgme-dev \
    squashfs-tools \
    libseccomp-dev \
    wget \
    pkg-config \
    git \
    cryptsetup-bin
```

Finally, download and build SingularityCE:

```
export VERSION=3.11.4 && \
    wget https://github.com/sylabs/singularity/releases/download/v${VERSION}/singularity-ce-${VERSION}.tar.gz && \
    tar -xzf singularity-ce-${VERSION}.tar.gz && \
    cd singularity-ce-${VERSION}

./mconfig && \
    make -C ./builddir && \
    sudo make -C ./builddir install
```

SingularityCE should now be available within WSL2 on your system.

## Getting an account on ARCHER2

### Sign up for a SAFE account

To sign up, you must first register for an account on [SAFE](https://safe.epcc.ed.ac.uk/) (our service administration web application):

If you are already registered on the ARCHER or Tier-2 SAFE you do not need to re-register. Please proceed to the next step.

1. Go to the [SAFE New User Signup Form](https://safe.epcc.ed.ac.uk/signup.jsp)
2. Fill in your personal details. You can come back later and change them if you wish. _**Note:** you should register using your institutional or company email address - email domains such as gmail.com, outlook.com, etc. are not allowed to be used for access to ARCHER2_
3. Click “Submit”
4. You are now registered. A single use login link will be emailed to the email address you provided. You can use this link to login and set your password.

### Sign up for an account on ARCHER2 through SAFE

In addition to your password, you will need an SSH key pair to access ARCHER2. There is useful guidance on how
to generate SSH key pairs in [the ARCHER2 documentation](https://docs.archer2.ac.uk/user-guide/connecting/#ssh-key-pairs).
It is useful to have your SSH key pair generated before you request an account on ARCHER2 as you can add it when 
you request the account

1. [Login to SAFE](https://safe.epcc.ed.ac.uk)
2. Go to the Menu "Login accounts" and select "Request login account"
3. Choose the `TBC` project “Choose Project for Machine Account” box and click "Next"
4.  Select the *ARCHER2* machine in the list of available machines
5.  Click *Next*
6.  Enter a username for the account and an SSH public
    key
    1.  If you do not specify an SSH key at this stage, your default
        key will be used (if you have one). For users who had an ARCHER
        account, the default key will be your ARCHER SSH key.
    2.  You can always add an SSH key (or additional SSH keys) using
        the process described below.
7.  Click *Request*

Now you have to wait for the course organiser to accept your request to register. When this has happened,your account will be created on ARCHER2.
Once this has been done, you should be sent an email. _If you have not received an email but believe that your account should have been activated, check your account status in SAFE which will also show when the account has been activated._ You can then pick up your one shot initial password for ARCHER2 from your SAFE account.

### Log into ARCHER2

You should now be able to log into ARCHER2 by following the [login instructions in the ARCHER2 documentation](https://docs.archer2.ac.uk/user-guide/connecting/#ssh-clients).



{% include links.md %}


