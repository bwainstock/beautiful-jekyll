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