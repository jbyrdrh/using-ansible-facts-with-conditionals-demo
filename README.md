Using Ansible Facts with Conditionals Demo

In this post: Learn how to push custom ansible_facts through a playbook to different managed hosts depending on system purpose: development, database, or webserver.

Environment overview
In my example environment, I have a control node system named controlnode running RHEL 9.4 and four managed nodes: rhel9-server1 and rhel9-server2, both running RHEL 9.4, and rhel8-server1 and rhel8-server2, both running RHEL 8.10.
