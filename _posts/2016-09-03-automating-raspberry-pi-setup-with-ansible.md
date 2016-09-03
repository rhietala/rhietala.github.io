---
layout: post
title: Automating Raspberry Pi setup with Ansible
---

![kuva]({{ site.url }}/assets/images/IMGP5898.jpg)

After installing an image to an SD card, and booting up a Raspberry Pi device with
the new operating system, it's time for the repetitive task of adding users,
installing software and configuring it. For one device this might be ok, but
what if you have to configure two or ten with the same parameters? And what if you
want to change to bigger SD cards with newer operating system image in your
tinkering environment?

[Ansible](https://www.ansible.com/) to the rescue. It is a tool for configuring and
managing computers that runs over an SSH connection and doesn't require any
client software to be installed on the managed computers. It also has a repository for
user-made configuration scripts, or *roles*, called
[Ansible Galaxy](https://galaxy.ansible.com/).

### Required hardware and software

*   Raspberry Pi(s) with Raspbian ([Installation instructions](https://www.raspberrypi.org/documentation/installation/installing-images/))
*   Internet connectivity and IP address of the Raspberry Pi
*   Ansible ([Installation instructions](http://docs.ansible.com/ansible/intro_installation.html))
*   Clone of [raspberry-ansible](https://github.com/rhietala/raspberry-ansible) repository

I have used the [Raspbian Jessie lite](https://www.raspberrypi.org/downloads/raspbian/)
image on [Raspberry Pi 2 model B](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/),
but all models and both versions of the Raspbian image should work.

### Repository file structure

    ./bootstrap.yml
    ./hosts
    ./keys/
    ./roles/firewall/
    ./roles/raspbian_bootstrap/

*   `bootstrap.yml` is an Ansible *playbook* that contains instructions about which
roles are run on which hosts, and with which variables.
*   `hosts` has a list of hosts where the configurations are run into.
*   `keys` is a directory for SSH keys for passwordless access to the Raspberries.
*   `roles/firewall` contains tasks for setting up firewall.
*   `roles/raspbian_bootstrap` has lots of tasks for setting up user, hostname, apt,
    etc. This is a fork of [debian_boostrap](https://github.com/HanXHX/ansible-debian-bootstrap)
    by [Emilien Mantel](https://twitter.com/hanxhx_) with minor modifications to
    suit Raspbian.

### Configuring device(s)

The devices are listed in `hosts` file with the following simple syntax:

    raspberry1 ansible_host=192.168.100.37

where first word is the desired hostname for the device (used as an ansible
alias and changed to be the device's hostname). Connection will be made to
the IP address given in `ansible_host` always.

Multiple devices can be listed with each on their own line.

### Configuring SSH keys

SSH keys can be generated with for example
[Atlassian's instructions](https://confluence.atlassian.com/bitbucketserver/creating-ssh-keys-776639788.html).
This will give you a private and a public key (`id_rsa` and `id_rsa.pub` respectively).

Public key has to be copied to the Raspberry, and these Ansible scripts will do
it as long as the file location is correct in `bootstrap.yml`. Private key is
used when connecting to the device: after public key is set up correctly to the device,
password authentication is no longer needed and the keys are used instead.

I have used keys generated specifically for this task and put them in this repository's
`keys` directory, but they can be in your home directory as well. In this case, these
configuration parameters in `bootstrap.yml` have to be changed:

{% raw %}
```
ansible_ssh_private_key_file: "keys/id_rsa"
dbs_ssh_pubkey: "{{ lookup('file', 'keys/id_rsa.pub') }}"
```
{% endraw %}

### Running the playbook

The playbook is run with the following command:

    ansible-playbook -i hosts bootstrap.yml --ask-pass

On the first run to a fresh Raspbian install with no SSH keys configured, it
will ask for pi-user password (`raspberry` by default). Then it will ask to what
this password will be changed into:

    $ ansible-playbook -i hosts bootstrap.yml --ask-pass
    SSH password:
    Enter new password for user pi:

    PLAY [all] *********************************************************************
    ...

### Testing the changes

After all, there aren't that many visible changes as these roles will do just
basic housekeeping that probably would have been left undone unless it was automated.

Anyways, now you should be able to log in with the SSH key and see the changed hostname
for example:

    $ ssh pi@192.168.100.37 -i keys/id_rsa
    ...
    pi@raspberry1:~ $

Or you can check for NTP status:

    $ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
    +dsl-kpobrasgw1- .PPS.            1 u  233  256  377   42.722   -2.835   1.288
    *ntp2.tdc.fi     .PPS.            1 u   64  256  367   15.933    0.240   1.363
    -ns2.posiona.net 192.36.133.17    2 u  190  256  377   15.099    0.149   0.637
    +87-92-47-29.bb. 192.36.144.23    2 u  259  256  377   13.248   -0.295   0.683

More interesting configurations in the next post!
