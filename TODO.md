* Write a man page.
* Add a regex filter (read from a file) to decide what to log and what to drop.
* Instead of requiring a fifo already exist, if the file doesn't already exist create the fifo.  Would need to modify the trap for SIGTERM in order to clean up the file.
* Figure out a way to check if the program is respawning too offten - this would indicate an error in the calling process and we don't want to just keep looping forever.
* Have an option to change UID and/or GID when running.  Alternatively, use setpriv to drop capabilities.
