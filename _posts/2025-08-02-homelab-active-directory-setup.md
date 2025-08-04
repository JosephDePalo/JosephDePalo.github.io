---
title: Consolidating Identity with Active Directory
date: 2025-08-02 12:00:00 -0400
categories: [Lab]
tags: [Home Lab]
image: /assets/img/b4fae878.png
---

*This post is a continuation of*
*[my previous home lab deployment post][homelab-deployment].*

Now that I've got a couple of endpoints on my home lab and some security
monitoring/collection tools deployed, I can turn my attention to something
that's been bugging me: having local user accounts on each machine. This is not
a very scalable solution and may be convenient at first, but that can quickly
change. Here's why.

## The Problem with not Having Centralized Identity Management

I have an admin account on my Wazuh and Splunk machines called `king-frieza`.
This is supposed to be a single account for a single admin, but these accounts
are completely separate besides the fact that they have the same username and
password.

Let's say I needed to change the password for `king-frieza`. This
would require me to login to each machine that `king-frieza` is on (`LWAZ01`
and `LSPL01` in this case) and change the password for the local user account.
This may not be a terrible process for 2 machines, but what if I had 20
instead? A small change such as that could take an hour or more. There's also
no guarantees that there is any consistency between the accounts or guarantees
that an account is who they say they are; anybody with sudo or root privileges
can spin up an account called `king-frieza`.

## Centralized IAM with Active Directory

This is where a tool like Microsoft's **Active Directory (AD)** comes into
play. AD is a centralized IAM solution used for managing users, computers,
authentication, access control, and more. Some of the key benefits of using AD
include:

- Single Sign-On (SSO)
- Simplified User Management
- Centralized Enforcement of Security Controls
- Central Logs for Access

Right now, we are mainly concerned with how AD simplifies user management. We
will explore some of its many other features in future posts. Using AD, I can
have a single `king-frieza` account that can be used across multiple machines
and services. AD is now our one stop shop for anything to do with managing this
account.

## Deploying AD in the Home Lab

Alright, so I want AD on the network. How do I do it? First off, we'll need to
stand up a **domain controller (DC)**. This is a server that serves as a
central authority in a **domain**, which is a logical grouping of network
objects. We will be creating a new domain and joining the machines
in my home lab to it.

Domain controllers are simply instances of Windows Server with the *Active*
*Directory Domain Services* feature enabled. For my lab, I spun up a VM running
Windows Server 2022 and enabled this feature, promoting it to the domain
controller `LDC01`.

Since we do not have any domains to join, we need to create a **forest**. A
forest is just a logical grouping of one or more domain controllers and
represents a whole AD instance; everything is contained within the forest.
After going through the creation process, we're prompted to name our domain.
For mine, I chose `jd.lab`.

Once all of this configuration is done, we have officially setup a domain and
domain controller. Our domain controller looks a little lonely though. It's
time to give it some friends (or subjects).

## Joining a Windows Endpoint to the Domain

We'll start with `LWIN01`, the Windows 10 endpoint. Joining a domain on Windows
is not a very complicated process, since AD was practically made for it.

First off, we need to set the machine's primary DNS server to `LDC01`. DNS is
what AD is built off of and it wouldn't work without it. `LWIN01` needs to be
able to resolve records contained only on `LDC01`, so it is necessary that this
change be made or we wouldn't be able to find the `jd.lab` domain.

Next, we join the domain by finding the *Access work and school* setting in
the Windows 10 settings menu and clicking *Connect*. This will prompt us to
enter a domain name and then ask for us to login with an account.

At this point, we don't have any AD accounts besides administrator. We want to
get `bob-brown` back, so let's head back to our domain controller to recreate
him. Using the *Active Directory Users and Computers* menu, we can provision a
new user `bob-brown` and use him to join the domain on `LWIN01`. `bob-brown` is
not an admin, so we'll keep him as a standard user.

After rebooting, we'll now have the option to login to `jd\bob-brown`, our new
Active Directory user.

## Joining a Linux Endpoint to the Domain

Joining `LUBU01` to the domain is going to be a tad more involved since there
aren't native Ubuntu tools for communicating with AD.

1. Install the required packages with the following command:

   ```bash
   sudo apt install realmd sssd sssd-tools adcli packagekit samba-common-bin
   ```

2. Modify `/etc/systemd/resolved.conf` to set the domain controller as the
   primary DNS server.

   ```conf
   [Resolve]
   DNS=10.0.0.35
   Domains=jd.lab
   ```

   Restart the DNS resolver with the following command:

   ```bash
   sudo systemctl restart systemd-resolved
   ```

3. Run `realm discover jd.lab` to confirm DNS is properly configured and the
   domain is discoverable. Join the domain with the following command:

   ```bash
   sudo realm join jd.lab -U 'JD\\Administrator'
   ```

   Now when we run `realm list`, we should see information for our domain.
4. Create an AD User `john-doe` to replace our local user. We can then allow
   him to login to `LUBU01` by running the following command:

   ```bash
   sudo realm permit jd\\john-doe
   ```

This is what it takes to join a Linux machine to a domain. When we login to
this account, however, you may notice that we don't have any home directory. To
have home directories be created for any AD user that logs into the machine, a
few additional steps are required.

1. Edit `/etc/sssd/sssd.conf` to have the following lines:

   ```conf
   [domain/jd.lab]
   fallback_homedir = /home/%u
   use_fully_qualified_names = False
   ```

2. Ensure `libpam-mkhomedir` is installed and then run `sudo pam-auth-update`.
   Check the *Create home directory on login* box and exit.

Now when we login to `jd\john-doe`, we should see that we are in our home
directory `/home/john-doe`.

The process for adding `LWAZ01` and `LSPL01` to the domain are the same as
adding `LUBU01` since they all run on Linux.

## Conclusion

Now we've consolidated *most* of the user credentials in the home lab, but
there is still work to be done. pfSense, Wazuh, and Splunk all still have their
own authentication methods that work independent of any centralized IAM
solution. This is something I intend to cover in the future as I explore AD
integrations into these services or setting up an alternative SSO system.

[homelab-deployment]: <{% post_url 2025-07-30-homelab-deployment %}>
