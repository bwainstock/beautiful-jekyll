---
layout: post
published: false
title: Ansible Callback Plugins
date: '2017-10-10'
---
1. Problem
  - Users coming back to me after deployment that multiple systems had issues
  - Were these errors with my playbook or with something else affecting
    - Puppet
    - Update me script
  - Realized I had no logging for my Ansible runs
  
My team and I deploy and configure hundreds of servers every week for internal teams across the world.  Our install process has evolved over the years into a system works, except when it doesn't.  Below is a general walkthrough of how an empty hard drive in a pizza box becomes a fully functioning application server.

Depending on the hardware configuration and operating system required for our server, a custom kickstart file installs our OS.  The `%post` section of the kickstart includes a hundreds of lines of custom shell script to configure things like disk partitions, environment variables, network interfaces, and other basic system level configuration.  This section also downloads a bash script that is set to run upon first boot which performs all sorts of configuration, including running Puppet.

Puppet then runs and enforces all of its requirements.  More package installation, NFS mounts, security settings, fstab and cron entries, and more are setup through Puppet.

Once Puppet has finished doing its job, an admin runs an Ansible playbook to configure application specific settings.  This could include application users and groups, packages, disk configurations, NFS shares, and more.  For a select few applications, this is the part of the system that I'm in charge of.  So what's the point of all of this back story?

The other day, an application owner came to me after I configured 50 or so servers for his team.  He reported to me that the home directory for his application user was not configured properly, even though the post install Ansible playbook which controls this had been run.  It was in this moment that I realized I have no logs to verify that this Ansible task was run successfully.  As any good ops person would do, I set out to roll my own Ansible logging using the [callback plugins feature](https://docs.ansible.com/ansible/devel/plugins/callback.html) I'd read about in the past.

2. Solution
  - Implement logging
    - Log start, summary, and individual tasks per host
    - Easily searchable
    - Utilize ansible callback plugins
    - Log playbook and ad hoc commands
3. Resources
  - http://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html
  - https://docs.ansible.com/ansible/devel/plugins/callback.html
  - https://github.com/ansible/ansible/tree/devel/lib/ansible/plugins/callback
4. Implementation
5. Usage
6. Future
  - CLI client
