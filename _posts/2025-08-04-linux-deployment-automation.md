---
title: Automating Linux Deployments with Ansible
date: 2025-08-04 09:00:00 -0400
categories: [Lab]
tags: [Home Lab]
image: /assets/img/a24dbf11.jpg
---

*This post is a continuation of*
*[my previous post on Active Directory][ad-post].*

After finishing up my Active Directory deployment and joining `LUBU01` to the
domain, I now had two more machines to join: `LSPL01` and `LWAZ01`. The process
for joining these would be the exact same as joining `LUBU01`, but I'd have to
repeat the steps manually for each one and that seems like a process that can
be made more efficient with automation. What if I decide to spin up a new
machine, `LUBU02`? The setup time for a machine is drawn out by the fact
that I'll have to go through the whole joining process for every single Linux
box in my lab.

## Ansible

This is where the beauty of automation shines brightest. The steps to join a
domain and perform the setup necessary aren't too complicated and can be done
by a script. A system administrator could then just log on to the machine, run
the script, and voila, the machine is part of the domain.

This approach is far better than the manual approach, but it still has some
problems:

- We have to be careful to ensure the script is **idempotent**. If it is run a
  second time on a machine, it shouldn't cause any unintended changes.
- A Bash or Python script can be complex, filled with conditionals, and have
  hard coded edge cases.
- We have to manually deal with any error checking and system checks.
  If we have multiple machines, we have to go run the script on each.
- Measures will have to be taken to ensure that any secrets used for the script
  are stored securely.

When it comes to configuration management for Linux, there is a clear winner
that solves all of these problems: **Ansible**. Ansible is an open source
configuration management and automation tool. It's very useful for managing,
configuring, or deploying multiple systems in a repeatable, declarative, and
agentless way. To address the above concerns:

- Ansible is designed for idempotency, and most actions have this property.
- Ansible playbooks are defined as simple YAML files which are quite easy to
  read and understand in a short amount of time.
- Ansible deals with error checking and system checks in a robust manner.
  Everything is done via SSH so we do not even have to log on to our target
  machines.
- Ansible Vault securely handles secrets for us.

Before we work on defining an Ansible playbook to automate our linux deployment
process, there are a few design considerations to think of.

## Playbook Considerations

First off, let's think about privileges and users. After joining an endpoint to
a domain, it still isn't much good until somebody can log on to it. This
requires us to perform a `realm permit` operation. So far we've been permitting
individual users to access machines, but this solution isn't very scalable.
`king-cold`, as an admin, needs access to all the boxes on the network so he
can configure and manage them. If we had `king-cold` added to every box, it may
work for a little while, but what happens when `king-cold` leaves and a new
user, `frieza`, takes over his job as admin? Now we'll have to go through every
box, withdraw permissions for `king-cold`, and grant `frieza` the same
permissions. There is a much better way to do this.

**Role-Based Access Control (RBAC)** is a security model that restricts system
access based on the role of individual users. With Active Directory, security
groups serve as our "roles" and we can associate privileges with these instead
of users. In *Active Directory Users and Computers*, I'll make a new security
group called *Engineers* and add `king-cold` to it. As part of our playbook,
we'll have the whole /Engineers/ group be permitted to access Linux machines
rather than just an individual user. This solves our problem.

Next, we should consider how sudo privileges will be managed. Let's say there
is a junior engineer on the support team and they can't exactly be trusted to
have full control over all systems yet. How can we be more specific about who
gets sudo privileges on all our Linux machines? The answer is quite simple: we
can just create another security group in AD called *Sudoers*. This gives us a
more granular way to manage access and further shows the beauty of RBAC.

## Ansible Setup

Ansible will access our Linux machines via SSH, and using user accounts for
Ansible would be a credential management disaster. For our purposes Ansible
will not only need machine access, but also sudo privileges. The best way to do
this is to set up service accounts on each machine called `ansible` with sudo
privileges that do not require a password. Such an account, however, is
essentially equivalent to root and thus a major vulnerability if not properly
secured. SSH key-based authentication helps with this. Instead of having to
enter a password to authenticate over SSH, a public and private key pair can be
used instead and is considered the better option in most cases. We can simply
generate a key pair on our machine, copy the contents of the public key to
`~/.ssh/authorized_keys`, and then we should be able to use our private key to
login to a remote machine without needing a password.

Now that we've got authentication figured out, we actually have to go and
create the `ansible` users. This brings us back to our original problem of
having to perform manual work on each machine, and we can't use Ansible because
Ansible doesn't have access yet. All is not lost, however. We can create a
script that will bootstrap the `ansible` user and just run that on each
machine. It's not the best solution, but it's the best we've got for now and
will only have to be done once on every machine. After the user is created,
Ansible can handle the rest.

*See my [GitHub repository][scripts-folder] for this script.*

## Ansible Vault

In order to join a domain, you need the credentials of an account with
permissions to join machines. Storing such information directly in a playbook
would be insecure. If I wanted to share this playbook with someone else, it
would also mean giving them the credentials to that account. Instead, we can
use Ansible Vault to securely store the credentials for our account. A vault is
essentially an encrypted variable file used for Ansible playbooks.

We can create a vault by using the `ansible-vault create` command and can
specify a password file to use (this isn't necessary, but it saves us from
having to enter a password every time we want to run a playbook with vault
variables. Keep this password file out of your project). When we put variables
in here and then `cat` out the file, you'll just see some metadata and a bunch
of numbers. This is because secrets in Ansible Vault are never stored in
plain text, so even if this file was accidentally shared or committed to a
repository, no secrets would be leaked.

In our vault, we can put our account's username and password.

```yaml
domain_user: "Administrator"
domain_user_password: "abc123"
```

## The Playbook

*The files discussed below can be found on my [GitHub repository]
[playbook-repo].*

Now for our actual playbook and our project structure. I won't go into too much
detail since many of the steps are the same as what was described in my
previous post (now in YAML form). My directory structure can be seen below:

```text
.
├── ansible.cfg
├── ansible_key
├── ansible_key.pub
├── inventory
│   └── hosts.ini
├── playbooks
│   └── linux-join-domain.yml
├── scripts
│   └── create_ansible_user.sh
├── vars
│   └── linux-join-domain.yml
└── vault
    └── secrets.yml
```

We'll be focusing on `playbooks/linux-join-domain.yml`, but it is worthwhile to
mention `inventory/hosts.ini` and `vars/linux-join-domain.yml`.

`inventory/hosts.ini` is an Ansible inventory file. It specifies the hosts we
will be running our playbook on and some relevant information for logging into
them (though `ansible.cfg` can abstract some of this information away). My
`hosts.ini` file looks like this:

```ini
[linux]
lubu01 ansible_user=ansible
lspl01 ansible_user=ansible
lwaz01 ansible_user=ansible
```

`vars/linux-join-domain.yml` just contains some variables relevant to my
playbook that do not have to be kept secret in my vault.

```yaml
domain_name: jd.lab
permitted_group: engineers
dns_server: 10.0.0.35
sudo_group: sudoers
sudoers_file: /etc/sudoers.d/domain_sudoers
```

For my actual playbook, I will only include a snippet here. The rest can be
found on [this repository][playbook-repo].

```yaml
---
- name: Join Linux machine to Active Directory domain
  hosts: linux
  become: true
  vars_files:
    - ../vars/linux-join-domain.yml
    - ../vault/secrets.yml
  tasks:
    - name: Ensure required packages are installed
      apt:
        name:
          - realmd
          - sssd
          - sssd-tools
          - adcli
          - packagekit
          - samba-common-bin
        state: present
        update_cache: yes

    - name: Set DNS in /etc/systemd/resolved.conf
      ini_file:
        path: /etc/systemd/resolved.conf
        section: "Resolve"
        option: DNS
        value: "{{ dns_server }}"
        no_extra_spaces: true
        mode: "0600"

# The rest can be found on GitHub.
```

This playbook runs on the `linux` group, *becomes* root, uses the variable
files specified, and performs the following list of tasks. Ansible has many
different modules to make performing certain actions easier, like `apt` for
installing packages on Debian-based machines and `ini_file` for editing files
in the INI format.

When we run the below command, Ansible will log into the machines specified in
our inventory and perform all of the tasks we laid out in our playbook. It'll
also show us if there were any issues running any of the tasks and if there
were problems connecting to the machine.

```bash
ansible-playbook playbooks/linux-join-domain.yml --vault-password-file ~/.ansible_vault_pass.txt
```

## Conclusion

Now we have an automated way to join all of our Linux endpoints to the domain
and the infrastructure to manage all of their configurations going forwards.
Ansible is a very powerful tool and I've barely scratched the surface of it. As
I continue building out my home lab, I'm looking forward to learning more about
its intricacies and use cases.

[ad-post]: <{% post_url 2025-08-02-homelab-active-directory-setup %}>
[scripts-folder]: <https://github.com/JosephDePalo/homelab-playbooks/tree/main/scripts>
[playbook-repo]: <https://github.com/JosephDePalo/homelab-playbooks/tree/main>
