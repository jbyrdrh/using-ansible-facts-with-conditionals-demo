**Using Ansible Facts with Conditionals Demo**

**In this post:**
- Learn how to push custom `ansible_facts` through playbooks to different managed hosts depending on each server's purpose: development, database, or webserver.

**Environment overview**

In my example environment, I have a control node system named `controlnode` running RHEL 9.4 and four managed nodes: `rhel9-server1` and `rhel9-server2`, both running RHEL 9.4, and `rhel8-server1` and `rhel8-server2`, both running RHEL 8.10.
Also note that the virtual machines are in 3 different networks.

**Cloning the GitHub Demo Repository**

*NOTE: The files provided in this repo are tailored specifically for my Red Hat test environment. It is important to note that you will need to modify hostnames, filenames, IP addresses, etc. as appropriate for your test environment.*

I recommend [adding your controlnode SSH key to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account?tool=webui) in order to clone the repository.

Once the SSH key has been added and GitHub is configured on `controlnode` with your [email address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address) and [username](https://docs.github.com/en/get-started/getting-started-with-git/setting-your-username-in-git), then clone the reposistory in the following manner:
~~
[jbyrd@controlnode]$ git clone git@github.com:jbyrdrh/using-ansible-facts-with-conditionals-demo.git
Cloning into 'using-ansible-facts-with-conditionals-demo'...
remote: Enumerating objects: 3507, done.
remote: Counting objects: 100% (3507/3507), done.
remote: Compressing objects: 100% (2360/2360), done.
remote: Total 3507 (delta 733), reused 3455 (delta 700), pack-reused 0
Receiving objects: 100% (3507/3507), 3.54 MiB | 14.22 MiB/s, done.
Resolving deltas: 100% (733/733), done.
~~


**The ansible.cfg and inventory files**

I used the following `ansible.cfg` file which has the ansible user running the commands on the managed hosts, and I referenced the Ansible Galaxy community collection, `community.general`, which is used in the demo. Also, the `privilege_escalation` section includes become=True, so it does not need to be added to the playbooks.

~~
[defaults]
inventory= ./inventory
remote_user=ansible
collections_path=./collections:/usr/share/ansible/collections

[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
~~

The `inventory` file for this demo is as follows:

~~
[webservers]
rhel8-server1
rhel8-server2

[databases]
rhel9-server1

[developer_servers]
rhel9-server2
~~

**Installing Collections from Ansible Galaxy**

Two collections from Ansible Galaxy are required for this demo: [ansible.posix](https://galaxy.ansible.com/ui/repo/published/ansible/posix/) and [community.general](https://galaxy.ansible.com/ui/repo/published/community/general/).

Run the following commands to create the `collections` subdirectory  and install the appropriate collections inside of this newly created directory.
~~
[jbyrd@controlnode using-ansible-facts-with-conditionals-demo]$ mkdir collections

[jbyrd@controlnode using-ansible-facts-with-conditionals-demo]$ ansible-galaxy collection install community.general -p collections/  --force
Starting galaxy collection install process
...
community.general:9.0.1 was installed successfully

[jbyrd@controlnode using-ansible-facts-with-conditionals-demo]$ ansible-galaxy collection install ansible.posix -p collections/  --force
Starting galaxy collection install process
...
ansible.posix:1.5.4 was installed successfully
~~

*NOTE: If the collection is already installed, you can force a reinstall of the collection with with the `--force ` flag as shown.*

**Playbook #1:   1_push_facts.yml**
The first playbook verifies the `/etc/ansible/facts.d/` directory exists on each managed node, and it then pushes custom facts to this directory.

*NOTE: This playbook will fail if you have not tailored the project files to align with your test environment.*

~~
---
- name: Copy custom_facts.fact to servers
  hosts: all
  become: yes

  tasks:
    - name: Ensure /etc/ansible/facts.d directory exists
      ansible.builtin.file:
        path: /etc/ansible/facts.d
        state: directory
        mode: '0755'

    - name: Copy custom_facts.fact to /etc/ansible/facts.d
      ansible.builtin.copy:
        src: host_vars/{{ inventory_hostname }}.fact
        dest: /etc/ansible/facts.d/custom_facts.fact
        mode: '0644'
~~

**Playbook #2:   2_run_group_specific_tasks.yml**

The 2nd playbook includes tasks that call the playbooks in the `tasks` directory, `databases.yml`, `developer_servers.yml`, `hardening.yml`, and `webservers.yml`, according to how each server is grouped in the `inventory` file.

~~
---
- name: Execute group-specific tasks
  hosts: all

  tasks:
    - name: Include tasks for webservers group
      include_tasks: tasks/webservers.yml
      when: "'webservers' in group_names" 

    - name: Include tasks for databases group
      include_tasks: tasks/databases.yml
      when: "'databases' in group_names"

    - name: Include tasks for developer_servers group
      include_tasks: tasks/developer_servers.yml
      when: "'developer_servers' in group_names"

    - name: Perform Security hardening on Secure VLAN (10.8.50.X)
      include_tasks: tasks/hardening.yml
      when: ansible_facts.default_ipv4.network is defined and ansible_facts.default_ipv4.network.startswith("10.8.50") 
~~

**Bonus: The undo scripts**

This `undo` directory contains scripts to **undo** some of the key changes made by the playbooks in case you want to run through the demo two or more times (without the playbooks skipping most tasks). This is a time-saving alternative to reverting snapshots on the managed nodes (if those exist for you) to a time before the playbook changes.
