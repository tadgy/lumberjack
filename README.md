# Lumberjack
In its simplest form, lumberjack is a pipe logger for Apache HTTPd (and other daemons).

Lumberjack can be used as the logger for piped logfiles using the HTTPd `CustomLog` directive - processing each log line and writing it out to the correct - per `VirtualHost` - log file.  It may also be used by other daemons capable of writing log files to a pipe, or via a FIFO.

In contrast to having one logger process per `VirtualHost` (or a hard coded log file for each `VirtualHost`, as is the norm), lumberjack can be set as the pipe logger in the global section of the *httpd.conf* file, and - with an appropriate `LogFormat` specifier - will split off each log line for a different `VirtualHost` and write it to a per `VirtualHost` log file based upon a user defined template.

Lumberjack may also be used in 'raw' mode in combination with the HTTPd `ErrorLog` directive to log errors, either on a per server or per `VirtualHost` basis to a log file based upon a user supplied template.

### Main features
  * Automatic log file compression (using a user specifiable compressor) after log files are rotated.
  * Request flushing/syncing of log files after each write (though this is not recommended).
  * Reading of log lines from a FIFO rather than stdin - allowing use with other daemons such as ftpd and rsyncd.
  * Automatic creation of a symbolic link to the currently active log file.
  * 'raw' logging mode, for use with HTTPd's ErrorLog and with other daemons such as ftpd and rsyncd.
  * User specified umask settings for directories and files.
  * Log file names based upon a strftime() encoded template, optionally including the HTTPd VirtualHost identifier.


## Usage
A full set of currently supported options can be obtained using `lumberjack -h` - this will always show the most up to date list of options.

The usages detailed below are the simplest form of the logger - options such as `-ca`, `-cc`, `-f`, `-j`, `-ud` and `-uf` which modify the behaviour of lumberjack will not be included in the examples.  Using those options should be fairly self-explanitory after reading `lumberjack -h` output.

### As a global Apache HTTPd 'CustomLog' pipe logger
In this mode of operation, you do not need to configure a `CustomLog` for each `VirtualHost` you use in your configuration.  Instead, the `CustomLog` directive is placed in the global section of the *httpd.conf* and will become effective for every `VirtualHost` (and the main server configuration, if used).

This is the recommended mode of operation for Apache HTTPd, as it only uses a single lumberjack process to handle logs for every `VirtualHost` defined in the configuration.

Before configuring lumberjack as the `CustomLog` pipe logger in apache HTTPd, it is necessary to add a custom `LogFormat` specifier for use with lumberjack.  This new `LogFormat` specifier is the same as the standard Apache 'Combined' log format - just with the addition of the '%v' specifier.  lumberjack may be used with any set of `LogFormat` specifiers, as long as the first specifier is always `%v`.  You can equally use lumberjack with the 'Common' log format identifier, or a custom one of your own specification.
For the purposes of this usage example, we will be using the 'Combined' format.

Create a new `LogFormat` entry (in this case named "VHostCombined") in the global section of *httpd.conf*, based upon the "Combined" format already given as an example in the default *httpd.conf*; but with the addition of the `%v` specifier for lumberjack:

```
    LogFormat    "%v %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\""    VHostCombined
```



### As a per VirtualHost 'CustomLog' pipe logger


### As an Apache HTTPd 'ErrorLog' pipe logger


### As a generic 'raw' pipe logger (for use with other daemons)


### With a FIFO (for use with other daemons)

