Yakuake session management
--------------------------

Requires: yakuake, qdbus (libqt4-dbus), pwdx (procps), ps (procps)

Tested on: Ubuntu Maverick (10.10) with Python 2.6 and Yakuake 2.9.7

Does not support old versions of yakuake that use DCOP instead of DBUS.

ysess is a script that gathers as much info as possible from a running yakuake instance and saves it out in INI format.  It can then take this same INI file as input and (destructively! \*) reconstruct the yakuake session.

\* As part of the reconstruction process, ysess destroys all current tabs in yakuake before restoring from the INI file.

Currently, ysess will save the following information:

 * tab order
 * tab title
 * active tab
 * per-tab working directory
 * per-tab active command

Per-tab active command is supported to allow resuming long-running commands, like ``top``.

    $ ysess --help
    Usage: ysess [options]

    Save and load yakuake sessions.  Settings are exported in INI format.  Default
    action is to print the current setup to stdout in INI format.

    Options:
      -h, --help            show this help message and exit
      -i FILE, --in-file=FILE
                            File to read from, or "-" for stdin
      -o FILE, --out-file=FILE
                            File to write to, or "-" for stdout
      --force-overwrite     Do not prompt for confirmation if out-file exists

To use in your crontab -- and I have no clue why this works -- prepend the command with ``DISPLAY=:0.0``:

    * * * * * DISPLAY=:0.0 ysess -o ~/yakuake.ini --force-overwrite

In the future...
================

I hope to add some features:

 * Save per-tab shell environment (so, for example, virtualenvs can be restored)
 * Allow a whitelist or blacklist for which active commands to save
 * Maybe, eventually, DCOP support for users stick with older yakuake versions

