**Using Ansible Facts with Conditionals Demo**

**In this post:**
Learn how to push custom `ansible_facts` through a playbook to different managed hosts depending on system purpose: development, database, or webserver.

**Environment overview**
In my example environment, I have a control node system named `controlnode` running RHEL 9.4 and four managed nodes: `rhel9-server1` and `rhel9-server2`, both running RHEL 9.4, and `rhel8-server1` and `rhel8-server2`, both running RHEL 8.10.
Also note that the virtual machines are in 3 different networks.

**The ansible.cfg and inventory files**
I used the following `ansible.cfg` file which has the ansible user running the commands on the managed hosts, and I referenced the Ansible Galaxy community collection, community.general, which is used in the demo. Also, the privilege_escalation section includes become=True, so it does not need to be added to the playbooks.

~~~
[defaults]
inventory= ./inventory
remote_user=ansible
collections_path=./collections:/usr/share/ansible/collections

[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
~~~

The `inventory` file for this demo is as follows:

~~~
[webservers]
rhel8-server1
rhel8-server2

[databases]
rhel9-server1

[developer_servers]
rhel9-server2
~~~
