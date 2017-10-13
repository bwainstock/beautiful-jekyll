---
layout: post
published: true
title: Ansible Callback Plugins
date: '2017-10-10'
---
## What kind of ops team doesn't have logging anyways?

My team and I deploy and configure hundreds of servers every week for internal teams across the world.  Our install process has evolved over the years into a system that works, except when it doesn't.  Below is a general walkthrough of how an empty hard drive in a pizza box becomes a fully functioning application server.

Depending on the hardware configuration and operating system required for our server, a custom kickstart file installs our OS.  The `%post` section of the kickstart includes hundreds of lines of custom shell script to configure things like disk partitions, environment variables, network interfaces, and other basic system level configuration.  This section also downloads a bash script that is set to run upon first boot which performs all sorts of configuration, including running Puppet.

Puppet runs and enforces all of its requirements.  Extra package installation, NFS mounts, security settings, fstab and cron entries, and more are setup through Puppet.

Once Puppet has finished doing its job, an admin runs an Ansible playbook to configure application specific settings.  This could include application users and groups, packages, disk configurations, NFS shares, and more.  For a select few applications, this is the part of the system that I'm in charge of.  So what's the point of all of this back story?

The other day, an application owner came to me after I configured 50 or so servers for his team.  He reported to me that the home directory for his application user was not configured properly, even though the post install Ansible playbook which controls this had been run.  It was in this moment that I realized I have no logs to verify that this Ansible task was run successfully.  As any good ops person would do, I set out to roll my own Ansible logging using the [callback plugins feature](https://docs.ansible.com/ansible/devel/plugins/callback.html) I'd read about in the past.

## Well, what are we going to do about it?

Now that I knew I needed a logging solution, I needed to think about what I wanted out of it.  My initial requirements looked a bit like this:

1. Utilize Ansible's callback plugin
2. Log playbooks and ad-hoc commands
3. Record information about the start and end of each playbook run as well as information about each task included in the playbook
4. Easily parsable so I can search


Based on requirement #1, I poked around Ansible's [documentation](http://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html) for developing callback plugins.  Callback plugins are just python scripts which implement a `CallbackBase` class made up of a number of methods.  These methods are called at different times throughout a playbook run process.  What happens when these methods are called is up to the user.  This takes care of requirement #3.

Thankfully, Ansible being a fully developed tool, requirement #2 is easy to achieve with a simple configuration setting.  With one configuration change, you can have the `ansible` command use the same callback plugin as `ansible-playbook`.

Finally, requirement #4 could lead me down a number of paths.  However, after watching a talk from Richard Hipp, the developer of SQLite, a number of years ago, I've been a pretty firm supporter of the tool for single instance applications.  SQLite not only allows me to contain all of the data in one file, but it opens up the world of SQL allowing for easy search queries.

## Putting it all into action

The `CallbackBase` class defines a number of methods that are called at various points throughout a task or playbook run.  The name of the method generally explains when it will be called:

```python
v2_on_any(self, *args, **kwargs)
v2_on_file_diff(self, result)
v2_playbook_on_cleanup_task_start(self, task)
v2_playbook_on_handler_task_start(self, task)
v2_playbook_on_import_for_host(self, result, imported_file)
v2_playbook_on_include(self, included_file)
v2_playbook_on_no_hosts_matched(self)
v2_playbook_on_no_hosts_remaining(self)
v2_playbook_on_not_import_for_host(self, result, missing_file)
v2_playbook_on_notify(self, result, handler)
v2_playbook_on_play_start(self, play)
v2_playbook_on_setup(self)
v2_playbook_on_start(self, playbook)
v2_playbook_on_stats(self, stats)
v2_playbook_on_task_start(self, task, is_conditional)
v2_playbook_on_vars_prompt(self, varname, private=True, prompt=None, encrypt=None, confirm=False, salt_size=None, salt=None, default=None)
v2_runner_item_on_failed(self, result)
v2_runner_item_on_ok(self, result)
v2_runner_item_on_skipped(self, result)
v2_runner_on_async_failed(self, result)
v2_runner_on_async_ok(self, result)
v2_runner_on_async_poll(self, result)
v2_runner_on_failed(self, result, ignore_errors=False)
v2_runner_on_file_diff(self, result, diff)
v2_runner_on_no_hosts(self, task)
v2_runner_on_ok(self, result)
v2_runner_on_skipped(self, result)
v2_runner_on_unreachable(self, result)
v2_runner_retry(self, result)


```

Since I've decided on my requirements already, I can pick and choose which methods I'd like to use.  So I've settled on the following skeleton that I need to implement:

```python
from ansible.plugins.callback import CallbackBase

class SQLiteCallbackModule(CallbackBase):

	def __init__(self):
    	pass

	def v2_playbook_on_start(self, playbook):
    """
    Run at the start of a playbook run
    """
    	pass
        
	def v2_playbook_on_stats(self, stats):
    """
    Run at the end of a playbook run
    """
    	pass

	def v2_runner_on_failed(self, result, **kwargs):
    """
    Run when a task fails
    """
    	pass
	
    def v2_runner_on_ok(self, result, **kwargs):
    """
    Run when a task succeeds
    """
    	pass
	
    def v2_runner_on_skipped(self, result, **kwargs):
    """
    Run when a task is skipped
    """
    	pass
        
	def v2_runner_on_unreachable(self, result, **kwargs):
    """
    Run when a host is unreachable
    """
    	pass
```

That's all that's needed for ansible to do its part.  However, we need to tell ansible what to do during these method calls.  When each of these methods is run, the goal is to have a SQLite database populated with a log message.  Generally, we'll need to log a message during three events:

1.  Playbook start
2.  Playbook finish
3.  Task finish (ok, fail, skipped, unreachable)

I could write raw SQL within the callback plugin, but I would really prefer not to.  I find it's easier to abstract the SQL with an ORM.  An easy and straightforward python ORM is [peewee](https://github.com/coleifer/peewee).  Peewee allows me to define a Model class with database fields defined in python.  Each Model will store information about the specific message, depending on the type.  Along with type-specific information, I'll store the actual type along with a session ID, which will allow me to correlate tasks to playbook runs.

```python
# models.py
"""
Models to be used for ansible-logs
"""
import os
from datetime import datetime
from peewee import Model, CharField, DateTimeField, DoubleField, SqliteDatabase

DB_NAME = os.getenv("SQLITE_DB", os.path.expanduser("~/.ansible.db"))
DB = SqliteDatabase(DB_NAME)


class AnsibleStart(Model):

    status = CharField(null=True)
    host = CharField(null=True)
    session = CharField(null=True)
    ansible_type = CharField(null=True)
    ansible_playbook = CharField(null=True)
    created_date = DateTimeField(null=True, default=datetime.now)

    class Meta:
        database = DB


class AnsibleFinish(Model):

    status = CharField(null=True)
    host = CharField(null=True)
    session = CharField(null=True)
    ansible_type = CharField(null=True)
    ansible_playbook = CharField(null=True)
    ansible_playbook_duration = DoubleField()
    ansible_host = CharField(null=True)
    ansible_result = CharField(null=True)
    created_date = DateTimeField(null=True, default=datetime.now)

    class Meta:
        database = DB
        

class AnsibleTask(Model):

    status = CharField(null=True)
    host = CharField(null=True)
    session = CharField(null=True)
    ansible_type = CharField(null=True)
    ansible_playbook = CharField(null=True)
    ansible_host = CharField(null=True)
    ansible_task = CharField(null=True)
    ansible_result = CharField(null=True)
    created_date = DateTimeField(null=True, default=datetime.now)

    class Meta:
        database = DB

```

These three classes are all I need to create three SQLite tables within a single database file.  Now when my callback plugin runs, I need to make sure my table exists and then add a log message.

```python
import json
import os
import socket
import uuid
from datetime import datetime

from ansible.plugins.callback import CallbackBase

from models import DB as db
from models import (AnsibleFinish, AnsibleImport, AnsibleStart, AnsibleTask)


class SQLiteCallbackModule(CallbackBase):
	# Ansible uses these environment variables
	CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'aggregate'
    CALLBACK_NAME = 'sqlite'
    CALLBACK_NEEDS_WHITELIST = True
    
    def __init__(self):
        super(CallbackModule, self).__init__()
        self.hostname = socket.gethostname()
        self.session = str(uuid.uuid1()) # Session ID
        self.errors = 0
        self.start_time = datetime.utcnow() # Used to calculate total elapsed time

	def v2_playbook_on_start(self, playbook):
    """
    Run at the start of a playbook run
    """
        db.connect()
        try:
            db.create_table(AnsibleStart)
        except OperationalError:
            pass

        self.playbook = os.path.join(playbook._basedir, playbook._file_name)
        data = {
            'status': "OK",
            'host': self.hostname,
            'session': self.session,
            'ansible_type': "start",
            'ansible_playbook': self.playbook,
        }
        ansible_start = AnsibleStart(**data)
        ansible_start.save()
        db.close()
        
	def v2_playbook_on_stats(self, stats):
    """
    Run at the end of a playbook run
    """
        db.connect()
        try:
            db.create_table(AnsibleFinish)
        except OperationalError:
            pass

        end_time = datetime.utcnow()
        runtime = end_time - self.start_time
        for host in stats.processed.keys():

            if self.errors == 0:
                status = "OK"
            else:
                status = "FAILED"

            data = {
                'status': status,
                'host': self.hostname,
                'session': self.session,
                'ansible_type': "finish",
                'ansible_playbook': getattr(self, 'playbook', 'NO PLAYBOOK'),
                'ansible_playbook_duration': runtime.total_seconds(),
                'ansible_host': host,
                'ansible_result': json.dumps(stats.summarize(host)),
            }
            ansible_finish = AnsibleFinish(**data)
            ansible_finish.save()
            db.close()

	def v2_runner_on_failed(self, result, **kwargs):
    """
    Run when a task fails
    """
        db.connect()
        try:
            db.create_table(AnsibleTask)
        except OperationalError:
            pass

        data = {
            'status': "FAILED",
            'host': self.hostname,
            'session': self.session,
            'ansible_type': "task",
            'ansible_playbook': getattr(self, 'playbook', 'NO PLAYBOOK'),
            'ansible_host': result._host.name,
            'ansible_task': result._task,
            'ansible_result': self._dump_results(result._result)
        }
        self.errors += 1
        ansible_task = AnsibleTask(**data)
        ansible_task.save()
        db.close()
	
    def v2_runner_on_ok(self, result, **kwargs):
    """
    Run when a task succeeds
    """
        db.connect()
        try:
            db.create_table(AnsibleTask)
        except OperationalError:
            pass
            
        data = {
            'status': "OK",
            'host': self.hostname,
            'session': self.session,
            'ansible_type': "task",
            'ansible_playbook': getattr(self, 'playbook', 'NO PLAYBOOK'),
            'ansible_host': result._host.name,
            'ansible_task': result._task,
            'ansible_result': self._dump_results(result._result)
        }
        ansible_task = AnsibleTask(**data)
        ansible_task.save()
        db.close()
        
    def v2_runner_on_skipped(self, result, **kwargs):
    """
    Run when a task is skipped
    """
        db.connect()
        try:
            db.create_table(AnsibleTask)
        except OperationalError:
            pass

        data = {
            'status': "SKIPPED",
            'host': self.hostname,
            'session': self.session,
            'ansible_type': "task",
            'ansible_playbook': getattr(self, 'playbook', 'NO PLAYBOOK'),
            'ansible_task': result._task,
            'ansible_host': result._host.name
        }
        ansible_task = AnsibleTask(**data)
        ansible_task.save()
        db.close()
        
	def v2_runner_on_unreachable(self, result, **kwargs):
    """
    Run when a host is unreachable
    """
        db.connect()
        try:
            db.create_table(AnsibleTask)
        except OperationalError:
            pass

        data = {
            'status': "UNREACHABLE",
            'host': self.hostname,
            'session': self.session,
            'ansible_type': "task",
            'ansible_playbook': getattr(self, 'playbook', 'NO PLAYBOOK'),
            'ansible_host': result._host.name,
            'ansible_task': result._task,
            'ansible_result': self._dump_results(result._result)
        }
        ansible_task = AnsibleTask(**data)
        ansible_task.save()
        db.close()
```

The callback plugin follows a simple pattern for each method.

1.  Connect to the database
2.  Ensure the table exists
3.  Parse and construct a log message
4.  Write to the database
5.  Close the database connection

## Putting the callback plugin into action

## How do I get informationn out of the database?
5. Usage
6. Future
  - CLI client
