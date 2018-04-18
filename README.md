# Lumberjack
In its simplest form, lumberjack is a pipe logger for Apache HTTPd and other daemons.

Lumberjack can be used as the logger for piped log files using the HTTPd `CustomLog` directive - processing each log line and writing it out to the correct - per `VirtualHost` - log file.  It may also be used by other daemons capable of writing log files to a pipe, or via a FIFO.

In contrast to having one logger process per `VirtualHost` (or a hard coded log file for each `VirtualHost`, as is the norm), lumberjack can be set as the pipe logger in the global section of the *httpd.conf* file, and - with an appropriate `LogFormat` specifier - will split off each log line for a different `VirtualHost` and write it to a per `VirtualHost` log file based upon a user defined template.

Lumberjack may also be used in 'raw' mode in combination with the HTTPd `ErrorLog` directive to log errors, either on a per server or per `VirtualHost` basis to a log file based upon a user supplied template.  'raw' mode also allows other daemons to log via lumberjack, either as a pipe logger or via a FIFO.

### Additional features
  * Automatic log file compression (using a user specifiable compressor) after log files are rotated.
  * Request flushing/syncing of log files after each write (though this is not recommended).
  * Reading of log lines from a FIFO rather than stdin - allowing use with other daemons such as ftpd and rsyncd.
  * Automatic creation of a symbolic link to the currently active log file.
  * 'raw' logging mode, for use with HTTPd's ErrorLog and with other daemons such as ftpd and rsyncd.
  * User specified umask settings for directories and files.
  * Log file names based upon a strftime() encoded template, optionally including the HTTPd VirtualHost identifier.


-----
## Command line and templates
```
lumberjack [options] <basedir> <template>
```
There are two mandatory arguments when using lumberjack; the base directory of the log file path (`<basedir>`) and the template of the log file path (`<template>`).  All other `[options]` are not required and only serve to modify the behaviour of lumberjack.

The base directory (`<basedir>`) is the path to the root of where the log files should be written.  This could be, for example, the root of the tree of sites which are virtualhosted, each with their own `logs/` directory.  Or it could be a seperate directory tree specifically for log files.  The `<basedir>` is pretty flexible and highly dependant upon your local filesystem layout.

Personally, I use a layout resembling:
```
    /data/sites/                              - This would be my `<basedir>`.
                <virtualhost name>/           - This is the name of the site served by the `VirtualHost`.
                                   html/      - Where the web content is stored.
                                   logs/      - The logs directory for the site.
```
Under the `logs/` directory, I have files written in the format: `access-log-YY-MM` - where `YY` is the two digit year, and `MM` is the two digit month.  An example full logfile path would therefore be: `/data/sites/afterdark.org.uk/logs/access-log-18-04`

To accomplish this format for a log file, the `<template>` would need to be set to: `{}/logs/access-log-%y-%m`.  `{}` appearing anywhere (even multiple times) in the template will be replaced with the site name as taken from `VirtualHost` identifier in the *httpd.conf* (see usage below for how to configure this for use with lumberjack).  The `%y-%m` are [`strftime()`](http://man7.org/linux/man-pages/man3/strftime.3.html) encoded expansions which would yeild the current two digit year, and current two digit month.

The templaing features of lumberjack is one of its most powerful features.  It can be used to create any free form log file path under the `<basedir>`.


-----

## Usage
A full set of currently supported options can be obtained using `lumberjack -h` - this will always show the most up to date list of options.

The usages detailed below are the simplest form of the logger - options such as `-ca`, `-cc`, `-f`, `-j`, `-ud` and `-uf` which modify the behaviour of lumberjack will not be included in the examples.  Using those options should be fairly self-explanitory after reading `lumberjack -h` output.

### As a global Apache HTTPd `CustomLog` pipe logger
In this mode of operation, you do not need to configure a `CustomLog` for each `VirtualHost` you use in your configuration.  Instead, the `CustomLog` directive is placed in the global section of the *httpd.conf* and will become effective for every `VirtualHost` (and the main server configuration, if used).

This is the recommended mode of operation for Apache HTTPd, as it only uses a single lumberjack process to handle logs for every `VirtualHost` defined in the configuration.

Before configuring lumberjack as the `CustomLog` pipe logger in apache HTTPd, it is necessary to add a custom `LogFormat` specifier for use with lumberjack.  This new `LogFormat` specifier is the same as the standard Apache 'Combined' log format - just with the addition of the '%v' specifier.  lumberjack may be used with any set of `LogFormat` specifiers, as long as the first specifier is always `%v`.  You can equally use lumberjack with the 'Common' log format identifier, or a custom one of your own specification.
For the purposes of this usage example, we will be using the 'Combined' format.

Create a new `LogFormat` entry (in this case named "VHostCombined") in the global section of *httpd.conf*, based upon the "Combined" format already given as an example in the default *httpd.conf*; but with the addition of the `%v` specifier for lumberjack:

```
    LogFormat    "%v %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\""    VHostCombined
```

Then locate the `CustomLog` directive in the *httpd.conf*.  Or you may need to add this if there is none already present.

The format for the `CustomLog` directive for use with lumberjack is:
```
    CustomLog    "|/path/to/lumberjack -l access.log -z /path/to/basedir {}/logs/access-%Y-%m.log"    VHostCombined
```

Here, the `-l` and `-z` flags specify extra options to lumberjack - they can be omitted if you do not require them.  The template (final argument) is just an example based upon my personal layout for the directories that was given as an example earlier.


### As an Apache HTTPd `ErrorLog` pipe logger
Lumberjack can also be used to handle error logs generated by Apache HTTPd in almost the same way as the `CustomLog` example above.  The only difference is the addition of the `-r` option, to tell Lumberjack it should be working in 'raw' mode.

In 'raw' mode, no procesing of the log line is performed - the `VirtualHost` identifier is not extracted to be used as part of the log file name, and the log line is written verbatim.

In 'raw' mode, the template (and link name if specified) cannot include the special `{}` string, which is usually replaced with the `VirtualHost` identifier when in 'normal' mode - Lumberjack will not start if either of those options include a `{}` sequence.

```
    ErrorLog    "|/path/to/lumberjack -r /path/to/basedir/<virtualhost name>/logs error-%Y-%m.log"
```

### As a generic 'raw' pipe logger (for use with other daemons)


### With a FIFO
Using lumberjack with daemons that do not support pipe logging is as simple as setting up a FIFO for them to write their logs to, and telling lumberjack where to find it.

To do this, use the `-i` flag to tell lumberjack where to locate the FIFO for the daemon from which you want to log.  All other command line options are the same as detailed in the other examples..
