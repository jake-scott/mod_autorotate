# mod_autorotate configuration

## Description

| Description | Fully automated log rotation and compression |
| ------------| ------------ |
| Status | Apache 2 licensed |
| Module | Identifier	autorotate_module |
| Source File | mod_autorotate.c |
| Compatibility | Requires Apache 2.2 |


## Summary

The autorotate module is an automated log rotation and compression module for Apache 2.2+ with the following features:

- Support for all standard Apache log files
- Hourly, Daily, Weekly or Monthly rotation schedule with configrable offset into the desired period
- Configurable file naming
- Compresison of log files, with the ability to retain a configurable number of logs uncompresses
- Configuratble number of rotated log files to keep around


Log files defined by the following standard Apache directives can be controlled by mod_autorotate:

| Module | Directive |
| ------ | --------- |
| Core | ErrorLog |
| mod_log_config | CustomLog |
|| CookieLog
|| TransferLog
| mod_log_forensic | ForensicLog |
| mod_rewrite | RewriteLog |
| mod_cgi |ScriptLog |


Additional log files can be rotated, by either specifying the log file name, or the name of a module's directive that defines a log file.

## Download

- Version 1.2
   - https://github.com/jake-scott/mod_autorotate/tree/apache2.2

## Configuration

### Loading the module

The module can be loaded with the following Apache configuration line:
  `LoadModule autorotate_module   /usr/local/apache/libexec/mod_autorotate.so`

Loading the module will also enable it for all supported log files using default rotation parameters (see below)

### Configuration directives

####  AutorotateEnabled directive
| Description | Enable or disable the module |
| ----------- | ---------------------------- |
| Syntax | `AutorotateEnabled On|Off` |
| Default | `AutorotateEnabled On` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

This directive can be used to disable the autorotate module whilst leaving it loaded into the server.

Example:

`AutorotateEnabled Off`


#### AutorotatePeriod directive
| Description | The rotation period |
| ----------- | ---------------------------- |
| Syntax | `AutorotateEnabled Hourly|Daily|Weekly|Monthly` |
| Default | `AutorotateEnabled Monthly` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

This directive can be used to alter the period of rotation. By default rotations are performed at the start of a given period:
  | Period | Description |
  | ----------- | ---------------------------- |
  | Hourly | At the start of the hour |
  | Daily | At midnight |
  | Weekly | At midnight on Sundays |
  | Monthly | At midnight on the 1st day of the month |

Local time is used for scheduling.

An offset can be specified to alter this behaviour (see AutorotateOffset)

Example:

  `AutorotatePeriod Weekly`

#### AutorotateOffset directive
| Description | Offset from the start of period at which to rotate log files |
| ----------- | ---------------------------- |
| Syntax | `AutorotateOffset <seconds>` |
| Default | `AutorotateOffset 0` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

This directive allows an offset from the start of the rotation period to be specified. The offset can be negative, which will cause mod_autorotate to rotate a set time before the end of the period. This is most useful when AutorotatePeriod is monthly as months have a variable length.

To rotate logs one day before the end of the month:

```
AutorotatePeriod Monthly
AutorotateOffset -86400
```

To rotate logs on Tuesdays at 1am:

```
AutorotatePeriod Weekly
AutorotateOffset 90000
```

#### AutorotateFormat directive
| Description | Format of string appended to rotated log file names |
| ----------- | ---------------------------- |
| Syntax | `AutorotateFormat <formatstring>` |
| Default | `AutorotateFormat "%Y%m%d-%H:%M:%S"` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

Allows the rotated file name format to be controlled. formatstring is a *strftime(3)* time format string. The date string is appended to the log file names when they are rotated.

For exampe, the default format string results in error.log being rotated to files with names such as `error.log.20090503-00:00:00`.

#### AutorotateRestartMethod directive
| Description | Controls how Apache is restarted after a log rotate |
| ----------- | ---------------------------- |
| Syntax | `AutorotateRestarMethod full|graceful` |
| Default | `AutorotateRestartMethod graceful` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

Apache needs to be restarted after the log files have been rotated, in order for new files to be opened by the privileged parent process and inherited by the unprivileged child processes. This directive controls how mod_autorotate asks Apache to restart.

A full restart causes Apache to kill all of its children and will abandon any currently executing requests. A graceful restart signals all currently running children to terminate after they have finished processing the current request, and reloads the parent process which causes new children to be spawned to process new requests. The default graceful restart method will not interrups currently executing requests and is reccomended.

Example:

  `AutorotateRestartMethod graceful`


#### AutorotateKeep directive
| Description |Controls how many rotated log files are retained |
| ----------- | ---------------------------- |
| Syntax | `AutorotateKeep <integer>` |
| Default | `AutorotateKeep 0` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

This directive configures mod_autorotate to keep only a set number of rotated log files. Previous log files will be pruned at server startup or restart. The default setting (0) disabled the pruning of log files.

If compression is disabled and the current time is Fri 8th May 2009 17:05, then :

```
AutorotateRestartPeriod Hourly
AutorotateKeep 4
```

The errorlog files that exist at this time will be :

```
error.log.20090508-14:00:00
error.log.20090508-15:00:00
error.log.20090508-16:00:00
error.log.20090508-17:00:00
error.log
```

#### AutorotateCompressAfter directive
| Description | Controls after how many rotations a file is compressed
| ----------- | ---------------------------- |
| Syntax | `AutorotateCompressAfter <integer>` |
| Default | `AutorotateCompressAfter 1` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

This directive controls the compression of rotated log files. Previous log files will be rotated after the configured number of rotations has passed. The default setting (1) causes log files to be compressed as soon as they have been rotated. Values over one result in a number of uncompressed logs remaining on the file system. A value of zero disabled log compression.

If current time is Fri 8th May 2009 17:05, then :

```
AutorotateRestartPeriod Hourly
AutorotateKeep 4
AutorotateCompressAfter 2
```

The errorlog files that exist at this time will be :

```
error.log.20090508-14:00:00.gz
error.log.20090508-15:00:00.gz
error.log.20090508-16:00:00.gz
error.log.20090508-17:00:00
error.log
```

#### AutorotateCompressProgram directive
| Description | Sets a custom compression program
| ----------- | ---------------------------- |
| Syntax | `AutorotateCompressProgram <path>` |
| Default | `AutorotateCompressProgram /usr/bin/gzip` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

This directive enables a custom compression program. The program is called with the name of the log file that should be compresses, and should remain in the foreground until the compression has completed. It should exit with code 0 if successful, otherwise with a non-zero code. The program is expected to leave behind a log file with the supplied name plus the suffix specified by AutorotateCompressSuffix and is expected to remove the original file.

gzip and compress are examples of suitable compression programs.

Example:

```
AutorotateCompressProgram /usr/bin/compress
AutorotateCompressSuffix .Z
```


#### AutorotateCompressSuffix directive
| Description | Specifies the suffix added to files by the compression program
| ----------- | ---------------------------- |
| Syntax | `AutorotateCompressSuffix <string>` |
| Default | `AutorotateCompressSuffix .gz` |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.0 and later |

This directive allows the suffix appended to log file names by the compression program to be specified. It is important for this to be correct in order for mod_autorotate to sucessfully prune older files.

#### AutorotateCompressNiceLevel directive
| Description | Start log compression processes with a specified scheduling priority |
| ----------- | ---------------------------- |
| Syntax | `AutorotateCompressNiceLevel <int>` |
| Default | 0 |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.1 and later |

Apache will renice the log compression program to this level. This can be any number from -20 to 20, if Apache is running as root. It may be a number equal to or higher than Apache's nice level, is Apache is not running as root.

Example:

  ```
  AutorotateCompressNiceLevel 10
  ```

#### AutorotateAddLogDirective directive
| Description | Specifies an additional Apache directive to scan the  configuration for, that defines the location of a log file |
| ----------- | ---------------------------- |
| Syntax | `AutorotateAddLogDirective <string> [int]` |
| Default | None |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.1 and later |

This directive introduces a directive to scan for in the configuration, other than those exposed by the core modules. For instance, the mod_jk module exposes the JkLogFile directive.

The optional integer parameter specifies which word of the directive's value defines the name of the file.  The first word is assumed if not specified.

Example:

```
AutorotateAddLogDirective JkLogFile

## The MyLogFile directive defines the filename in the second word
MyLogFile compact  /var/log/foo.log
AutorotateAddLogDirective MyLogFile 2
```

#### AutorotateAddLogFile directive
| Description | Specifies an additional log file name to rotate |
| ----------- | ---------------------------- |
| Syntax | `AutorotateAddLogFile <string>` |
| Default | None |
| Context | server config |
| Override | None |
| Compatibility | mod_autorotate version 1.1 and later |

Request an additional log file to be rotated. This is intended for use where a log is written by an Apache extension, CGI script etc, where there is no configuration directive that defines its location.

Example:

```
  AutorotateAddLogFile logs/SM.log
```