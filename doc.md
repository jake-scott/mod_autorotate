# mod_autorotate internals

## Synopsys

mod_autorotate is a log rotation plugin module for Apache httpd v2.2.  It was written to aleviate the overhead of log management tasks in a large web farm, an environment where scheduling cleanup routines for tens of thousands of instances with a variaety of configurations is a challenge.  But its use is relevant to deployments of any size.

The module introspects the server's parsed configuration tree to determine the names of log files that the standard modules create.  It can be taught directives that third-party modules use to define a file that needs to be rotated.  Additionally it can be supplied with path names of files to rotate, to support the edge case where a module may use an alternate configuration mechanism to define its log file paths (eg. CA Siteminder).



## Configuration

Apache httpd modules are deployed as shared libraries.  A core configuration directive tells httpd the name of the shared library and the name of an exported symbol that points to a *module* structure.

The server locates the *autorotate_module* symbol exported by the declaration on line 44 in *mod_autorotate.c* (and specified by the server's *LoadModule* configuration directive).  The populated *module* structure  includes information about how it expects to be configured as well as a refernce to a function that registers hooks that will be called during various processing phases.

Configuration directives of the *auto_rotate* module are documented in README.md.

This module inserts hooks into two httpd processing phases - *monitor* and *open_logs*. *open_logs* is called by the main caretaker process during a server start or restart.  *monitor* is run by the caretaker process every 10 seconds in httpd 2.2.  The caretaker process is the parent of the request processing processes, and often runs as the root user.


## Operation

### Server restarts

Each restart of the server results in a new server *generation*.  Child process are registered along with their generation ID and other metadata, in httpd's central *scoreboard* - an area of shared memory accessible by every child.

The server supports a feature known as gracefully restart.  This allows active requests to continue being processed whilst a set of new request handler processes is spun up to process new requests.  The feature often results in children from several generations running at the same time.  This module takes graceful restarts into account to avoid log file corruption, and also supports non-graceful (full) restarts for configurations that include third-party modules that do not conform (eg. CA Siteminder).


### Rotation

Apache httpd calls the *open_logs* hook of each registered module at server startup or on restart.  mod_autorotate uses the *open_logs* hook to rotate logs.  It does this by renaming logs if the configured rotation peroid is due.  New log files are then created by other modules' *open_logs* handler.  Though is it safe to rename open log files, it is possible that child processes from a previous *generation* (in the case of graceful restart) are still writing to them, so compression must be done later when all previous-generation processes have exited.

The *open_logs* handler traverses the server's parsed configuration tree, looking for well-known and configured directives that refer to log files.  It runs before any other modules (by way of the *APR_HOOK_FIRST* parameter in `register_hooks`), and can therefore be sure that no current-generation processes have any files open that it rotates.  Any files that it renames are added to a compress queue for later processing.

At server startup, the module calculates the time that the next log rotation is due.  The *monitor* hook, which is run every 10 seconds by the care-taker process, restarts the server when that time has passed.  The server restart kicks off the next log rotation.


### Compression

As descibed above, Apache uses a process generation system as part of its graceful restart functionality.  Rotated log files are added to a compression queue at rotation time but only compressed when previous server generation processes have exited.

The module uses the server's *monitor* hook to track prevous server generations (as well as restarting the server when a rotation becomes due). The *monitor* hook is run every 10 seconds by the httpd caretaker process.  mod_autorotate uses this to check the generation of child processes in the *scoreboard*, and declines to run if old processes still exist.

Once the server is fully restarted, this module's *monitor* hook is responsible for  processing the old log compress queue.

The Apache Portable Runtime is used to manage compression processes.  The runtime provides an OS agnostic interface that allows the module to receive notifications via a call-back when compression processes have finished.  This is used to serialise the compression process and to register the children with the runtime (via the APR pool mechanism) so that they are killed if the server is terminated.


## Bugs etc

Configuration is per-server only;  it is not possible to configure the rotation of any lof files that might be written by modules supporting per directory configuration.

The module does not consideration daylight-savings events in its calculation of rotation periods and so it is possible that rotations could be missed or perfomed twice if such an event occurs close to the rotation time.

There are no tests!  Apache httpd modules are hard to test unless they are re-factored to avoid dependencies on the server internals -- which this module relies on heavily.  Separating the code into parts that are httpd specific and
APR specific would help grately.

This hasn't been tested on Windows or other non-unix OS's supported by Apache httpd / APR.  It likely needs some work in that regard, in particular the use of *setpriority(2)*, *getpid(2)* and *kill(2)*. 
