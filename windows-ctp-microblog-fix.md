# CTP Microblog Deployment Fix for Windows 10
```
Author: Herman Ho
Last edit: 09/06/2016
```

This is a document detailing the steps I've taken to get the ctp-microblog deployed on a Windows 10 environment.
Full disclosure, this took **hours** to troubleshoot, and I hope it works for others, but as with anything - YMMV.


There seems to be multiple issues here at play, preventing a successful deployment on Windows. The first has to do with the versions of Vagrant and VirtualBox. The following versions have been tested to work for me:

VirtualBox Version 5.1.4 r110228 (Qt5.5.1) - Newest as of 09/05/2016.

Vagrant Version 1.8.5 - Newest as of 09/05/2016

> The issue comes into play later on, due to some sort of symlink bug in VirtualBox. You can download the versions and install them on top of your current installations - both can be gracefully updated.

### File Modifications

This part gets a little hairy - multiple files need to be modified, although they are not *major* changes. The steps are:

1) Clone the repository to your host computer.
```
$ git clone https://github.com/medgardo/ctp-microblog.git
```
> If you already cloned the repo prior, scrap and start over.

2) Use your favorite text editor to modify the Vagrantfile and Ansible inventory file:

In `ctp-microblog/env/Vagrantfile` search for and replace with the following lines:
```
ansible.version = "2.0"
ansible.inventory_path = "inventories/local.ini"
```
The `ctp-microblog/env/inventories/local` file will also need to be updated with the correct extension,
either through the GUI or:
```
$ cd ctp-microblog/env/inventories/
$ mv local local.ini
```
> This is related to a bug in older versions of Ansible that treated inventory files as executable in Windows, rather than text/config files. The issue is described here:
https://github.com/ansible/ansible/issues/10068
and fixed here: https://github.com/ansible/ansible/pull/11643. The official changelog is here: https://github.com/ansible/ansible/blob/devel/CHANGELOG.md.

3) Next is the main Ansible playbook file:

Replace contents of `ctp-microblog/env/playbooks/main.yml` with:
```
- hosts: all
  remote_user: "{{ host_user|default('ubuntu') }}"
  become: yes
  become_method: sudo
  vars_files:
    - "{{ env }}/passwords.yml"
  roles:
    - initial

- hosts: postgres
  remote_user: "{{ app_user }}"
  become: yes
  become_method: sudo
  vars_files:
    - "{{ env }}/passwords.yml"
    - "{{ env }}/settings.yml"
  roles:
    - postgres

- hosts: app
  remote_user: "{{ app_user }}"
  become: yes
  become_method: sudo
  vars_files:
    - "{{ env }}/passwords.yml"
  roles:
    - blog
```

> This step only changes the names of some properties, due to deprecation in version 2.0. See here: http://docs.ansible.com/ansible/playbooks_intro.html.

4) Next, there is a step in the Ansible playbook that copies the `ctp-microblog/env/playbooks/roles/initial/files/etc/sudoers` file into your VM after initialization. This fails in Windows due to Windows line endings that are not recognized by the Ubuntu VM. To fix this, convert the file using Vagrant/Bash:
```
$ cd ctp-microblog/env/playbooks/roles/initial/files/etc
$ dos2unix sudoers
```


The above steps conclude the necessary file modifications.
The next steps are performed in the VM.


#### VM Modifications


1) Create and initialize the ctp-microblog VM using Vagrant/Bash:
```
$ cd ctp-microblog/env
$ vagrant up
```

> This takes a few minutes. If you followed the steps above, you should not get any errors.

2) SSH into the VM:
```
$ vagrant ssh
```

3) Next, it appears `nodemon` is not installed by default (?), so you will need to run:
```
$ sudo npm install -g nodemon
```

> Be sure to run with `sudo`.

4) Finally, start the ctp-microblog application:
```
$ cd /opt/apps/blog/
$ npm start
```

Switch to your host machine and type the following address into your browser:
```
192.168.33.10:3000
```

Voila! You should be presented with the ctp-microblog application!
