---
layout: post
title: Server configs and fabric
summary: Although fabric allows you to execute functions on multiple hosts at once, it lacks
    any sufficiently advanced instruments to store and apply server-specific options.
    Implementing such instrument is the main goal of this post.
---

[Fabric](http://fabfile.org/) is a deployment automation tool popular among python developers. Since
1.0 release it allows you to do pretty much anything you would normally do manually with ssh.

Fabric is simple and concentrates on providing an API, a number of utility functions and an easy
way to run tasks defined inside fabfiles. But although fabric allows you to execute functions
on multiple hosts at once, it lacks any sufficiently advanced instruments to store and apply
server-specific options. Implementing such instrument is the main goal of this post.

I've spent some time researching possible solutions to the following problem, however I should note
that I could've missed some obvious and easier solution or, since this was before the release of
fabric 1.0, such solution may have been introduced to fabric itself by the time of this writing.

## The Problem

Suppose we have three projects: ProjectA, ProjectB and ProjectC deployed on three corresponding
servers: ServerA, ServerB and ServerC. We want:

* to deploy each project to its own server with one shell command
* to deploy all projects at once with one shell command
* to run maintenance tasks on a subgroup of servers with one shell command

Fabric solves first problem really good, but it proposes no apparent solution for the rest.

Let's say our deployment workflow is similar for all projects and consists of two basic steps:
updating project source code from repository and restarting the server process. We could implement
such workflow with three fabric tasks:

{% highlight python %}
def update():
    # ...

def restart():
    # ...

def deploy():
    update()
    restart()
{% endhighlight %}

Writing these tasks for a single project is simple, however since we need them for each project it
would be nice to follow the DRY principle and implement them only once for use on any project with
similar deployment workflow.

And that's where things get tricky. We can't really implement the `update()` function without either
hardcoding the source code repository location and shell command to pull the update or some sort of
project config with repository location, which `update()` can address.

The same applies to `restart()` function. It's easy to write a function that restarts `apache`,
but what if one of our projects is served by `nginx` or `lighttpd`? How can we write a `restart()`
function that takes care of all the boilerplate of process restart(switching to correct user,
performing appropriate checks and so on) and restarts the correct process depending on the project
config.

When working with fabric, this is usually solved by defining environment-initializing functions
for each server:

{% highlight python %}
def server_a():
    env.hosts = 'a.example.org'
    # ...

def server_b()
    # ...
{% endhighlight %}

But this makes it impossible to run one task on multiple servers at once.
I.e. running
    
    $ fab server_a server_b deploy

won't do what you expect it to (if you expect it to deploy to two servers, that is).

Following is a fabfile template that will allow you to use server-specific configuration options
inside your task, an instruction on how to use it and a detailed explanation of implementation.

## Solution in a nutshell

1. Download base [fabfile.py](/static/code/fabfile.py) template. It requires fabric version 0.9 or later,
   includes a bunch of internal functions and defines one task: `s`.
   You can add your own tasks to this fabfile or, if you're using fabric version 1.0 or later, you
   can create [fabfile folder](http://fabric.readthedocs.org/en/1.0.1/usage/fabfiles.html),
   move it there and define your own tasks in other files inside that folder.
2. Add `@_setup` decorator to tasks that need additional server options.
3. Address all required server options as `env` attributes from your fabric tasks:

       @_setup
       def restart():
           if env.webserver == "apache":
               # restart apache
           else:
               # ...

   Attributes aren't namespaced and get automatically added to `env` by `_setup` decorator so you have
   to be careful not to overwrite any of the
   [builtin env arguments](http://docs.fabfile.org/en/1.0.1/usage/env.html).
4. In the root folder of your project create a file `server_config.yaml` or `server_config.json`
   if you prefer json syntax. To use YAML config file you need to have PyYaml
   installed (`pip install pyyaml` will generally suffice), while
   JSON requires either `simplejson` or python 2.6 with built-in `json` module.
5. Open created file and define server parameters for your servers. `host`
   is the only required parameter and it must contain server hostname:
    * YAML:

          server_a:
                host: a.example.com
                webserver: apache
                repository: git@example.com:project.git
          server_b:
                host: b.example.com
                webserver: nginx
                web_folder: /var/www/project
                ...

    * JSON:

          {
            "server_a":{
                "host": "a.example.com",
                "webserver": "apache",
                "repository": "git@example.com:project.git"
            },
            "server_b": {
                ...
            }
          }

6. Define server groups in the config file:
    * YAML:

          frontend-servers: [server_a, server_c, ...]

    * JSON:

          {
            ...
            "frontend-servers": ["server_a", "server_c", ...]
          }

After everything is set up, you can call 

     $ fab s

to list all available server configs, or
  
     $ fab s:server_a,server_b,frontend-servers task

to execute `task` once for server A, server B and members of `frontend-servers` group.

Since fabric looks for fabfile in parent directories you can move your file to the common parent
directory of your projects path and leave server config inside project directory. This way you
can use the same fabfile with different projects and keep server config together with other
project-related files.

If the above worked as expected and you aren't interested in any further explanations you can stop
reading now.

Having described the basic usage and configuration, let's move on to listing the initial
requirements and internal details of implementation.

## Detailed explanation

### Requirements

* Proposed solution must be **portable** in a sense that it should be possible to transfer the whole
  thing to other developers in one step and it should work on any machine with
  fabric already installed. This basically means patching fabric isn't an option.
* Conforming to the **DRY** principle means the user has to describe the task and list specific
  server options only once.
* Allow **any number of parameters** defined by user for each server.
* Allow **parameters of arbitrary type**: string, integer, lists and python dictionaries when
  needed.
* Introduce as **little overhead** as possible to both writing tasks and running them. Fabric is
  mostly about making the common thing easy and it should remain that way.
* Make it **easy to add** servers and config parameters to existing configuration files. 
* It should **scale** to any number of servers. Defining and running env initialization tasks may
  work for 2 or 3 servers, but it is basically unusable for 10 or more.
* Allow defining **server groups** the same way `env.roledefs` allows to arrange hosts in groups.
* **Allow to run one task on multiple servers** using only one shell command.
* Must be **compatible** with fabric's built-in methods to declare hosts and tasks that don't need
  this functionality.

### Fabric handling of env

Fabric keeps config options (including list of hosts and user info) inside global environment
variable `env`. 

`env` itself isn't very interesting: it's a simple dictionary subclass
([fabric/state.py](https://github.com/bitprophet/fabric/blob/master/fabric/state.py)) that allows to
access its keys as attributes. Inside it are stored all the settings that influence underlying ssh 
and internal settings. Full list of environment variables can be found in 
[fabric documentation](http://docs.fabfile.org/en/1.0.1/usage/env.html).
There are, however, two important details about env in
[fabric main loop](https://github.com/bitprophet/fabric/blob/master/fabric/main.py):

1. Once the task started running you can't in any way change its destination hosts.
   Any changes you make to `env.hosts` will influence the following tasks, not the current one.
   This is due to the fact that fabric iterates over internal `hosts` variable that
   is set once for each task before the task starts running.
2. Fabric deletes duplicates from host list by transforming it into set and back,
   so initial hosts order is lost during execution.

The second fact is interesting outside the context of our problem because it means that you
can't possibly know whether certain task will be first executed on server A or server B and so
you shouldn't rely on any specific order in your deployment routines.

### Config file syntax

Multiple-server config file is best represented by a dictionary: each server is represented by
server name (key) and parameters (value). Since we want to support arbitrary number of parameters,
we will represent each server config also as a dictionary, where parameter names will be keys, and
parameter values will be, well, values. We're using symbolic server names as keys, so each config
must provide server's hostname in the standard required parameter `host`.

We've mentioned the support for server groups before. Server group is basically a list of server
names stored under group name key. The reason for this is quite simple: if we have a number of
database servers and want to backup them all for example, instead of writing
    
    $ fab dbserver_1 dbserver_2 ... backup

each time, we would rather store the list of database servers in our config file under the
key "dbservers" and start the actual task with:

    $ fab dbservers backup

Server groups may be nested, for example:

{% highlight javascript %}
dbservers: [dbserver_1, dbserver_2]
...
allservers: [dbservers, webservers]
{% endhighlight %}

To write the actual config files we could use either YAML or JSON syntax: YAML is somewhat easier to
read and write by hand, but JSON usually won't require any additional packages since most developers
already have either `simplejson` or python 2.6.

We'll support both and choose the appropriate loader depending on the installed libraries and
available config file extension. Examples of server configs are shown in "Solution in a nutshell"
section.

### Parsing config files

We've come to the actual implementation details. First thing to do is to find the config file
and convert it from markup language to python data structure. We'll start with figuring out what
packages are available:

{% highlight python %}
YAML_AVAILABLE = True
try:
    import yaml
except ImportError:
    YAML_AVAILABLE = False

JSON_AVAILABLE = True
try:
    import simplejson as json
except ImportError:
    try:
        import json
    except ImportError:
        JSON_AVAILABLE = False
{% endhighlight %}

We're looking for `PyYAML` for YAML and either `simplejson` or built-in `json` module for JSON.
The actual function that transforms file to python dictionary:

{% highlight python %}
def _load_config(**kwargs):
    """Find and parse server config file.

    If `config` keyword argument wasn't set look for default
    'server_config.yaml' or 'server_config.json' file.

    """
    config, ext = os.path.splitext(kwargs.get('config',
    'server_config.yaml' if os.path.exists('server_config.yaml') \
                         else 'server_config.json'))

    if not os.path.exists(config + ext):
        print colors.red('Error. "%s" file not found.' % (config + ext))
        return {}
    if YAML_AVAILABLE and ext == '.yaml':
        loader = yaml
    elif JSON_AVAILABLE and ext =='.json':
        loader = json
    else:
        print colors.red('Parser package not available')
        return {}
    # Open file and deserialize settings.
    with open(config + ext) as config_file:
        return loader.load(config_file)
{% endhighlight %}

It starts by looking for one of the following:

1. Existing file with `config` keyword argument name.
2. Existing `server_config.yaml` file.
3. Existing `server_config.json` file.

If none are found, function will print error message and return empty dictionary. If one of the files
exists and the corresponding package is available, we parse the file and return python dictionary.

### Server selection task

Now we need to actually tell fabric to load and parse config files and a way to choose which servers
we'll be using for task execution. For this very reason we have created a single task `s`. You call
it before your own tasks and pass it server names the way you would pass arguments to any fabric
task:

    $ fab s:server_a,server_b,server_group mytask mytask2

Additionally, you can also specify the location of config file:

    $ fab s:server_a,server_b,config=servers.yaml mytask mytask2

Here's what it does:

{% highlight python %}
def s(*args, **kwargs):
    """Set destination servers or server groups by comma delimited list of names"""
    # Load config
    servers = _load_config(**kwargs)
    # If no arguments were recieved, print a message with a list of
    # available configs.
    if not args:
        print 'No server name given. Available configs:'
        for key in servers:
            print colors.green('\t%s' % key)

    # Create `group` - a dictionary, containing copies of configs for selected
    # servers. Server hosts are used as dictionary keys, which allows us to
    # connect current command destination host with the correct config. This
    # is important, because somewhere along the way fabric messes up the hosts
    # order, so simple list index incrementation won't suffice.
    env.group = {}
    # For each given server name
    for name in args:
        #  Recursive function call to retrieve all server records. If `name`
        # is a group(e.g. `all`) - get its members, iterate through them and
        # create `group` record. Else, get fields from `name` server record.
        # If requested server is not in the settings dictionary output error
        # message and list all available servers.
        _build_group(name, servers)


    # Copy server hosts from `env.group` keys - this gives us a complete list of
    # unique hosts to operate on. No host is added twice, so we can safely add
    # overlaping groups. Each added host is guaranteed to have a config record
    # in `env.group`.
    env.hosts = env.group.keys()
{% endhighlight %}

`s` takes as arguments a list of server names and, optionally, a keyword argument `config`,
containing the path to server config file. It starts with calling `_load_config` to parse config
file. If no server names were specified, `s` will print all server names found by `_load_config`.

We create a `group` dictionary in `env` and for each server name user has specified we call
`_build_group` function, which as we'll see soon, modifies `env.group` to store server configs. The
last thing we do in `s` is rewriting `env.hosts` with `env.group.keys()`, which will influence the
next task. At this point `env.hosts` will contain a list of server hosts that:

1. Belong to the servers that user has specified.
2. Belong to the servers that appear inside server groups that user has specified.

Let's look at `_build_group` function next to see how this list gets computed:

{% highlight python %}
def _build_group(name, servers):
    """Recursively walk through servers dictionary and search for all server records.

    """
    # We're going to reference server a lot, so we'd better store it.
    server = servers.get(name, None)
    # If `name` exists in servers dictionary we
    if server:
        # check whether it's a group by looking for `members`
        if isinstance(server, list):
            if fabric.state.output['debug']:
                    puts("%s is a group, getting members" % name)
            for item in server:
                # and call this function for each of them.
                _build_group(item, servers)
        # When, finally, we dig through to the standalone server records, we
        # retrieve configs and store them in `env.group`
        else:
            if fabric.state.output['debug']:
                    puts("%s is a server, filling up env.group" % name)
            env.group[server['host']] = server
    else:
        print colors.red('Error. "%s" config not found. \
                    Run `fab s` to list all available configs' % name)
{% endhighlight %}

Function starts by checking `servers` dictionary (which is just a parsed version of config file) for
existing `name` key, which indicates whether user specified a valid server name or not. If the key
was found, we check whether its value is a list, in which case we call `_build_group` for each item
from that list, or a dictionary, in which case we add server config to `env.group` dictionary using
`server["host"]` as key.

Thus, for each server name that is either directly specified by user or is a member of a server
group, specified by user, we store config dictionary inside `env.group[server['host']]`. This is
exactly the reason why `host` is required in server configs - since we need a way to create a host
list for fabric and a way to connect server on which we are executing tasks at the moment with
its config dictionary.

We could stop at this point, since we have everything we need for a working solution to the
specified problem:

    $ fab s:server_a,server_b mytask

will allow us to get current server options from `mytask`, e.g. to get `repository` option value for
current server we could do:

{% highlight python %}
repository = env.group[env.host_string]["repository"]
{% endhighlight %}

but this looks somewhat ugly, and doesn't really qualify as "making common thing easy".
There's a number of things we could do to simplify this, one of which is to define a helper function:

{% highlight python %}
def _get(key):
    return env.group[env.host_string].get(key, None)
{% endhighlight %}

This way, `_get("repository")` will return either the option value or `None` if option isn't defined
for the current server. This is much nicer to use, but one problem still remaining with this
solution is that it isn't compatible with the usual way to store server settings directly as attributes
of `env` (which is common inside special initialization tasks). This means that one task won't work out of the box
with both types of server initialization.

### Task setup decorator

To resolve the issue we'll need to rewrite all keys of current server configuration from
`env.group[env.host_string][key]` to `env.key` before each task run. Let's define a `_setup`
decorator, that will do just that:

{% highlight python %}
def _setup(task):
    """Copies server config settings from `env.group` dictionary to env variable.
    
    This way, tasks have easier access to server-specific variables:
        `env.owner` instead of `env.group[env.host]['owner']`

    """
    def task_with_setup(*args, **kwargs):
        # If `s:server` was run before the current command - then we should copy
        # values to `env`. Otherwise, hosts were passed through command line
        # with `fab -H host1,host2 command` and we skip.
        if env.group:
            for key,val in env.group[env.host].items():
                setattr(env, key, val)
                if fabric.state.output['debug']:
                    puts("[env] %s : %s" % (key, val))

        task(*args, **kwargs)
    return task_with_setup
{% endhighlight %}

Now all you have to do to make `env.key` access available inside your task is to decorate it with
`@_setup`. Generally this works just fine, however you should be wary of two things:

1. If you define a config option with the same name as one of the built-in `env` attributes, like
   `host_string` or `hosts`, things might break in unexpected ways.
2. Attributes aren't deleted after the task has finished. This means that if current server config
   doesn't define `key1` option, `env.key1` might still exist if one of previous server configs
   defined it.

## The whole thing
[Complete source code](/static/code/fabfile.py) contains everything described here, except the
`_get()` function example. Note that link above points to the most recent published version of the
script, while code snippets in this post may be outdated.
