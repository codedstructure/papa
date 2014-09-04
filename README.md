# papa

## Description
**papa** is a process kernel. It contains both a client library and a server
piece for creating sockets and launching processes from a stable parent process.

## Dependencies
It is tested under following Python versions:

- 2.6
- 2.7
- 3.3
- 3.4


## Installation
You can install it from Python Package Index (PyPI):

	$ pip install papa


## Purpose
Sometimes you want to be able to start a process and have it survive on its own,
but you still want to be able to capture the output. You could daemonize it
and pipe the output to files, but that is a pain and lacks flexibility when it
comes to handling the output.

Process managers such as circus and supervisor are very good for starting and
stopping processes, and for ensuring that they are automatically restarted when
they die. However, if you need to restart the process manager, all of their
managed processes must be brought down as well. In this day of zero downtime,
that is no longer okay.

Papa is a process kernel. It has extremely limited functionality and it has zero
external dependencies. If I've done my job right, you should never need to
upgrade the papa package. There will probably be a few bug fixes before it is
really "done", but the design goal was to create something that did NOT do
everything, but only did the bare minimum required. The big process managers can
add the remaining features.

Papa has 3 types of things it manages:

- sockets
- values
- processes

Here is what papa does:

- Create sockets and close sockets
- Set, get and clear named values
- Start processes and capture their stdout/stderr
- Allow you to retrieve the stdout/stderr of the processes started by papa
- Pass socket file descriptors and port numbers to processes as they start

Here is what it does NOT do:

- Stop processes
- Send signals to processes
- Restart processes
- Communicate with processes in any way other than to capture their output


## Sockets
By managing sockets, papa can manage interprocess communication. Just create a
socket in papa and then pass the file descriptor to your process to use it.
See the circus docs for a very good description of why this is so useful.

Papa can create Unix, INET and INET6 sockets. By default it will create an INET
TCP socket on an OS-assigned port.

You can pass either the file descriptor (fileno) or the port or a socket to a
process by including a pattern like this in the process arguments:

- $(socket.my_awesome_socket_name.fileno)
- $(socket.my_awesome_socket_name.port)


## Values
Papa has a very simple name/value pair storage. This works much like environment
variables. The values must be text, so if you want to store a complex structure,
you will need to encode and decode with something like the JSON module.

The primary purpose of this facility is to store state information for your
process that will survive between restarts. For instance, a process manager can
store the current state that all of its managed processes are supposed to be in.
Then if the process manager is restarted, it can restore its internal state,
then go about checking to see if anything on the machine has changed. Are all
processes that should be running actually running?


## Processes
Processes can be started with or without output management. You can specify a
maximum size for output to be cached. Each started process has a management
thread in the Papa kernel watching its state and capturing output if necessary.


## Naming
Sockets, values and processes all have unique names. A name can only represent
one item per class. So you could have an "aack" socket, an "aack" value and an
"aack" process, but you cannot have two "aack" processes.

All of the monitoring commands support a final asterix as a wildcard. So you
can get a list of sockets whose names match "uwsgi*" and you would get any
socket that starts with "uwsgi".

One good naming scheme is to prefix all names with the name of your own
application. So, for instance, the Circus process manager can prefix all names
with "circus." and the Supervisor process manager can prefix all names with
"supervisor.". If you write your own simple process manager, just prefix it with
"tweeter." or "facebooklet." or whatever your project is called.

Then if you need to have multiple copies of something, put a number after a dot
for each of those as well. For instance, if you are starting 3 waitress
instances in circus, call them "circus.waitress.0", "circus.waitress.1", and
"circus.waitress.2". That way you can query for all processes named "circus.*"
to see all processes managed by circus, or query for "circus.waitress.*" to
see all waitress processes managed by circus.


## Starting the kernel
There are two ways to start the kernel. You can run it as a process, or you can
just try to access it from the client library and allow it to autostart. The
client library uses a lock to ensure that multiple threads do not start the
server at the same time but there is currently no protection against multiple
processes doing so.

By default, Papa will start on port 20202. You can change this by specifying a
different port number or a path. By specifying a path, a Unix socket will be
used instead.

If you are going to be creating Papa client instances in many places in your
code, you may want to just call papa.set_default_port or papa.set_default_path
once when your application is starting and then just instantiate the Papa client
with no parameters.


## Telnet interface
Papa has been designed so that you can communicate with the process kernel
entirely without code. Just start the Papa server, then do this:

telnet localhost 20202

You should get a welcome message and a prompt. Type "help" to get help. Type
"help process" to get help on the process command.

The most useful commands from a monitoring standpoint are:

- sockets
- processes
- values

All of these can by used with no arguments, or can be followed by a list of
names, including wildcards. For instance, to see all of the values in the
circus and supervisor namespaces, do this:

values circus.* supervisor.*


## Creating a Connection
You can create either long-lived or short-lived connections to the Papa kernel.
If you want to have a long-lived connection, just create a Papa object to
connect and close it when done, like this:

```python
class MyObject(object):
    def __init__(self):
        self.papa = Papa()

    def start_stuff(self):
        self.papa.make_socket('uwsgi')
        self.papa.make_process('uwsgi', 'env/bin/uwsgi', args=('--ini', 'uwsgi.ini', '--socket', 'fd://$(socket.uwsgi.fileno)'), working_dir='/Users/aackbar/awesome', env=os.environ)
        self.papa.make_process('http_receiver', sys.executable, args=('http.py', '$(socket.uwsgi.port)'), working_dir='/Users/aackbar/awesome', env=os.environ)

    def close(self):
        self.papa.close()
```

If you want to just fire off a few commands and leave, it is better to use the
`with` mechanism like this:

```python
from papa import Papa

with Papa() as p:
    print(p.sockets())
    print(p.make_socket('uwsgi', port=8080))
    print(p.sockets())
    print(p.make_process('uwsgi', 'env/bin/uwsgi', args=('--ini', 'uwsgi.ini', '--socket', 'fd://$(socket.uwsgi.fileno)'), working_dir='/Users/aackbar/awesome', env=os.environ))
    print(p.make_process('http_receiver', sys.executable, args=('http.py', '$(socket.uwsgi.port)'), working_dir='/Users/aackbar/awesome', env=os.environ))
    print(p.processes())
```

This will make a new connection, do a bunch of work, then close the connection.


## Socket Commands
There are 3 socket commands.

# `sockets`
The `sockets` command takes a list of socket names to get info about. All of
these are valid:

- p.sockets()
- p.sockets('circus.*')
- p.sockets('circus.uwsgi', 'circus.nginx.*', 'circus.logger')

A dict is returned with socket names as keys and socket details as values.

# `make_socket`
All parameters are optional except for the name. To create a standard TCP socket
on port 8080, you can do this:

`p.make_socket('circus.uwsgi', port=8080)`

To make a Unix socket, do this:

`p.make_socket('circus.uwsgi', path='/tmp/uwsgi.sock')`

A path for a Unix socket must be an absolute path or make_socket will raise a
papa.Error exception.

You can also leave out the path and port to create a standard TCP socket with an
OS-assigned port. This is really handy.

If you call `make_socket` with the name of a socket that already exists, papa
will return the original socket if all parameters match, or raise a papa.Error
exception if some parameters differ.

See the `make_sockets` method of the Papa object for other parameters.

# `close_socket`
The `close_socket` command also takes a list of socket names. All of these are
valid:

- p.close_socket('circus.*')
- p.close_socket('circus.uwsgi', 'circus.nginx.*', 'circus.logger')


## Value Commands
There are 4 value commands.

# `values`
The `values` command takes a list of values to retrieve. All of these are valid:

- p.values()
- p.values('circus.*')
- p.values('circus.uwsgi', 'circus.nginx.*', 'circus.logger')

# `set`
To set a value, do this:

`p.set('circus.uswgi', value)`

# `get`
To retrieve a value, do this:

`value = p.get('circus.uwsgi')`

If no value is stored by that name, `None` will be returned.

# `clear`
To clear a value or values, do something like this:

- p.clear('circus.*')
- p.clear('circus.uwsgi', 'circus.nginx.*', 'circus.logger')

You cannot clear all variables so passing no names or passing '*' will raise
a papa.Error exception.


## Process Commands
There are 4 process commands:

# `processes`
The `processes` command takes a list of process names to get info about. All of
these are valid:

- p.processes()
- p.processes('circus.*')
- p.processes('circus.uwsgi', 'circus.nginx.*', 'circus.logger')

A dict is returned with process names as keys and process details as values.

# `make_process`

Every process must have a unique name and a command. All other parameters are
optional. The `make_process` method will return a dict that contains the pid of
the process.

The 'args' parameter should be a tuple of command-line arguments. If you have
only one argument, papa supports passing that as a string as a convenience.

You will probably want to pass 'working_dir'. If you do not, the working
directory will be that of the papa kernel process.

By default, 'stdout' and 'stderr' are captured so that you can retrieve them
with the `watch` command. By default, the 'bufsize' for the output is 1MB.

Valid values for 'stdout' and 'stderr' are papa.DEVNULL and papa.PIPE (the
default). You can also pass papa.STDOUT to 'stderr' to merge the streams. 

If you pass 0 for bufsize, not output will be recorded. Otherwise, bufsize can
be the number of bytes, or a number followed by 'k', 'm' or 'g'. So if you want
a 2 MB buffer, you can pass bufsize='2m', for instance. If you do not retrieve
the output quicky enough and the buffer overflows, older data is removed to make
room.

If you specify 'uid', it can be either the numeric id of the user or the
username. Likewise with 'gid'.

If you want to specify 'rlimits', pass a dict with rlimit names and numeric
values. Valid rlimit names can be found in the 'resources' module. Leave off the
'RLIMIT_' prefix. On my system, valid names are 'as', 'core', 'cpu', 'data',
'fsize', 'memlock', 'nofile', 'nproc', 'rss', and 'stack'.

`rlimit={'cpu': 2, 'nofile': 1024}`

The 'env' parameter also takes a dict with names and values. A useful trick for
'env' is to do `env=os.environ` to copy your environment to the new process.

If you want to run a Python application and you wish to use the same Python
executable as your client application, a useful trick is to pass sys.executable
as the 'cmd' and the path to the Python script as the first element of your
'args' tuple. If you have no other args, just pass the path as a string to
'args'.

`p.make_process('write3', sys.executable, args='executables/write_three_lines.py', working_dir=here, uid=os.environ['LOGNAME'], env=os.environ)`

The final argument that needs mention is `watch_immediately`. If you pass 'True'
for this, papa will make the process and return a watcher. This is effectively
the same as doing `p.make_process(name, ...)` followed immediately by
`p.watch(name)`, but it has one fewer round-trip communication with the kernel.
If all you want to do is launch an application and monitor its output, this is
a good way to go.

# `close_output`
If you do not care about retrieving the output of the exit code for a process,
you can use `close_output` to tell the papa kernel to close the output buffers
and automatically remove the process from the process list when it exits.

- p.close_output('circus.logger')
- p.close_output('circus.uwsgi', 'circus.nginx.*', 'circus.logger')

# `watch`
The `watch` command returns a `Watcher` object for the specified process or
processes. That object uses a separate socket to retrieve the output of
the processes it is watching.

Optimization Note: Actually, it hijacks the socket of your Papa
socket. If you issue any other commands to the Papa object that require a
connection to the kernel, the Papa object will silently create a new socket and
connect up for the additional commands. If you finish with the Watcher and the
Papa object has not been used for anything else yet, the socket will be
returned to the Papa object when you close the Watcher object. So if you launch
an application, use `watch` to grab all of its output until it closes, then use
the `processes` command to make see what is still running, all of that can occur
with a single connection.


## The Watcher object
