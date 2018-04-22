# Lumberjack
In its simplest form, lumberjack is a pipe logger for [Apache HTTPd](http://httpd.apache.org/) and other daemons.

Lumberjack can be used as the logger for piped log files using the HTTPd `CustomLog` directive - processing each log line and writing it out to the correct - per `VirtualHost` - log file.  It may also be used by other daemons capable of writing log files to a pipe, or via a FIFO.

In contrast to having one logger process per `VirtualHost` (or a hard coded log file for each `VirtualHost`, as is the norm), lumberjack can be set as the pipe logger in the global section of the *httpd.conf* file, and will - with an appropriate `LogFormat` specifier - split off each log line for a different `VirtualHost` and write it to a per `VirtualHost` log file based upon a user defined template.

Lumberjack may also be used in 'raw' mode in combination with the HTTPd `ErrorLog` directive to log errors to a log file based upon a user supplied template, either on a per server or per `VirtualHost` basis.  'raw' mode also allows other daemons to log via lumberjack, either as a pipe logger or via a FIFO.

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

The base directory (`<basedir>`) is the path to the root of where the log files should be written.  This could be, for example, the root of the tree of sites which are virtual hosted, each with their own `logs/` sub-directory.  Or it could be a seperate directory tree specifically for log files.  The `<basedir>` is pretty flexible and highly dependant upon your local filesystem layout.

Personally, I use a layout resembling:
```
    /data/sites/                              - This would be my `<basedir>`.
                <virtualhost name>/           - This is the name of the site served by the `VirtualHost`.
                                   cgi-bin/   - The virtual host\'s CGI binaries directory.
                                   html/      - The virtual host\'s web content.
                                   logs/      - The logs directory for the virtual host.
```
Under the `logs/` directory, I have files written in the format: `YYYY/MM/filename.log` - where `YYYY` is the year, and `MM` is the two digit month.  An example full logfile path would therefore be: `/data/sites/afterdark.org.uk/logs/2018/04/httpd-access.log`.

To accomplish this format for a log file, the `<template>` would need to be set to: `{}/logs/%Y/%m/httpd-access.log`.  The sequence `{}` appearing anywhere (even multiple times) in the template will be replaced with the site name as taken from `VirtualHost` site identifier from the *httpd.conf* (see usage below for how to configure this for use with lumberjack).  The `%Y` and `%m` are [`strftime()`](http://man7.org/linux/man-pages/man3/strftime.3.html) encoded expansions which would yeild the current year, and two digit month.

The templaing feature of lumberjack is one of its most powerful features.  It can be used to create any free form log file path under the `<basedir>`.


-----
## Usage
A full set of currently supported options can be obtained using `lumberjack -h` - this will always show the most up to date usage and options.

The usages detailed below are the simplest form of the logger - options such as `-ca`, `-cc`, `-f`, `-j`, `-ud` and `-uf` which modify the behaviour of lumberjack will not be included in the examples.  Using those options should be fairly self-explanitory after reading `lumberjack -h` output.

### As a global HTTPd `CustomLog` pipe logger
In this mode of operation, you do not need to configure a `CustomLog` for each `VirtualHost` you use in your configuration.  Instead, the `CustomLog` directive is placed in the global section of the *httpd.conf* and will become effective for every `VirtualHost` (and the main server configuration, if used).

This is the recommended mode of operation for Apache HTTPd, as it only uses a single lumberjack process to handle logs for every `VirtualHost` defined in the configuration.

Before configuring lumberjack as the `CustomLog` pipe logger in HTTPd, it is necessary to add a custom `LogFormat` specifier for use with lumberjack.  This new `LogFormat` specifier is the same as the standard Apache 'Common' or 'Combined' log formats - just with the addition of the '%v' specifier.  lumberjack may be used with any set of `LogFormat` specifiers, as long as the first item in the specifier is always `%v`.  

For the purposes of this usage example, we will be using the 'Combined' log format.

Create a new `LogFormat` entry (in this case named "VHostCombined") in the global section of *httpd.conf*, based upon the "Combined" format already given as an example in the default *httpd.conf*; but with the addition of the `%v` specifier for lumberjack:

```
    LogFormat    "%v %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\""    VHostCombined
```

Then locate the `CustomLog` directive in the *httpd.conf*.  Or you may need to add this if there is none already present.

The format for the `CustomLog` directive for use with lumberjack is:
```
    CustomLog    "|/path/to/lumberjack -l {}/logs/httpd-access.log -z /path/to/basedir {}/logs/%Y/%m/httpd-access.log"    VHostCombined
```

Here, the `-l` and `-z` flags are extra options to lumberjack - they can be omitted if you do not require them.  The template is just an example based upon my personal layout for the directories that was given as an example earlier; you can use any layout that suits your personal directory layout.


### As an Apache HTTPd `ErrorLog` pipe logger
Lumberjack can also be used to handle error logs generated by Apache HTTPd in almost the same way as the `CustomLog` example above.  The only difference is the addition of the `-r` option, to tell Lumberjack it should be working in 'raw' mode, and the `LogFormat` changes are not necessary.

Unlike in 'normal' mode, where lumberjack extracts the `VirtualHost` site identifier for use in the template (and removes that identifier from the lon entry written); in 'raw' mode, no procesing of the log line is performed - the `VirtualHost` identifier is not extracted to be used as part of the log file name, and the log line is written verbatim.

In 'raw' mode, the template (and link name if specified) cannot include the special `{}` string, which is usually replaced with the `VirtualHost` site identifier when in 'normal' mode - Lumberjack will not start if either of those options include a `{}` sequence.

Unfortunately, because Apache HTTPd does not allow custom formats for the error log (where we could include the site idetifier and only use one `ErrorLog` directive), we must use an `ErrorLog` per `VirtualHost`.  An example `ErrorLog` directive, based upon the examples given previously, would be:
```
    ErrorLog    "|/path/to/lumberjack -l httpd-error.log -r -z /data/sites/afterdark.org.uk/logs %Y/%m-httpd-error.log"
```

Here, the `<basedir>` includes the full path to the logs directory to be used, since it cannot be parsed from the log line itself.  The `<template>` and link name (`-l`) have also been changed to fit with the new `<basedir>`.

### As a generic 'raw' pipe logger (for use with other daemons)


### With a FIFO
Using lumberjack with daemons that do not support pipe logging is as simple as setting up a FIFO for them to write their logs to, and telling lumberjack where to find it.

To do this, use the `-i` flag to tell lumberjack where to locate the FIFO for the daemon from which you want to log.  All other command line options are the same as detailed in the other examples, but it is most likely that you'll also want to use the `-r` flag, to tell lumberjack not to process the log lines for HTTPd `VirtualHost` site identifiers.
