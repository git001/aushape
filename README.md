Aushape
=======

Aushape is a tool and a library for converting Linux audit log messages to
JSON and XML, allowing both single-shot and streaming conversion.

At the moment Aushape can output to stdout, a file, or syslog(3). The latter
outputs one document or event per message.

**NOTE**: Aushape is in early development stage and anything about its
interfaces and outputs can change. Use at your own risk.

Schemas
-------

Aushape output document schemas are still in flux, but the main idea is to
aggregate input records belonging to the same event into single output event
object/element, while keeping the naming and the structure as close to the
original audit log as possible.

A truncated JSON example:

    [
        {
            "serial"    : 123,
            "time"      : "2016-01-03T02:37:51.394+02:00",
            "host"      : "auditdtest.a1959.org",
            "records"   : {
                "syscall"   : {
                    "raw"       : "node=auditdtest.a1959.org type=SYSCALL ...",
                    "fields"    : {
                        "syscall"   : ["rt_sigaction","13"],
                        "success"   : ["yes"],
                        "exit"      : ["0"],
                        ...
                    }
                },
                "proctitle" : {
                    "raw"       : "node=auditdtest.a1959.org type=PROCTITLE ...",
                    "fields"    : {
                        "proctitle" : ["bash","\"bash\""]
                    }
                },
                ...
            }
            ...
        },
        ...
    ]


A truncated XML example:

    <?xml version="1.0" encoding="UTF-8"?>
    <log>
        <event serial="194433" time="2016-01-03T02:37:51.394+02:00" host="auditdtest.a1959.org">
            <syscall raw="node=auditdtest.a1959.org type=SYSCALL ...">
                <syscall i="rt_sigaction" r="13"/>
                <success i="yes"/>
                <exit i="0"/>
                ...
            </syscall>
            <proctitle raw="node=auditdtest.a1959.org type=PROCTITLE ...">
                <proctitle i="bash" r="&quot;bash&quot;"/>
            </proctitle>
            ...
        </event>
        ...
    </log>

There is a number of challenges, the main one being both the Linux kernel and
the Auditd code defining record structure and sometimes changing it from
version to version, without an official specification being there. Yet, we
have developed draft schemas for both [JSON](lib/aushape.json) and
[XML](lib/aushape.xsd), and will continue on improving them in collaboration
with Auditd developers.

We encourage you to simply try running aushape on your logs to see what the
output structure is like.

Dependencies
------------

Aushape uses the Auparse library (a part of the Auditd package) to parse audit
logs. The development version of this library needs to be installed before
building Aushape. It is available in "audit-libs-devel" package on Fedora and
RHEL, and "libauparse-dev" package on Debian-based systems.

Building
--------

If you'd like to build Aushape from the Git source tree, you need to first
generate the build system files:

    autoreconf -i -f

After that you need to follow the usual configure & make approach:

    ./configure --prefix=/usr --sysconfdir=/etc && make

Installing
----------

You can install Aushape with the usual `make install`:

    sudo make install

Usage
-----

### Single-shot

For one-shot conversions simply use the `aushape` program. E.g. to convert an
audit.log to the default JSON:

    aushape audit.log

or explicitly:

    aushape -l json audit.log

To convert to XML:

    aushape -l xml audit.log

To write output to a file:

    aushape audit.log > audit.json

or:

    aushape -f audit.json audit.log

### Live

You can also use Aushape as an Auditd's Audispd plugin to convert messages as
they are generated by the system. However, since Audispd doesn't support
supplying more than two (unquoted) command-line arguments to plugins, you'll
have to write a little wrapper script to configure Aushape appropriately and
specify *that* to Audispd as the program to run.

If you would like your audit events converted to JSON and sent to syslog, one
event per message, you can write this wrapper and put it, for example, into
`/usr/bin/aushape-audispd-plugin`:

    #!/bin/sh
    exec /usr/bin/aushape -l json --events-per-doc=none --fold=all -o syslog

Don't forget to make it executable.

You can then add it to Audispd configuration by putting this into
`/etc/audisp/plugins.d/aushape.conf`:

    active = yes
    direction = out
    path = /usr/local/bin/aushape-audispd-plugin
    type = always
    format = string

After Auditd is restarted, the events should be logged to syslog with
"authpriv" facility and "info" priority (you can change these with more
command-line options to `aushape`). Beside the Systemd's journal, if you also
use rsyslogd with default configuration, they would end up in
`/var/log/secure` on Fedora and RHEL, and in `/var/log/auth.log` on
Debian-based systems.

**NOTE**: Some audit events can be large. For example the execve events can be
in the order of megabytes for very long command lines. Most logging servers
will drop long messages silently. Make sure your audit configuration
only logs events which are not too long and/or configure your logging server
to accept longer messages.

We are working on implementing event size limit and truncation in Aushape, so
you can at least see when an event was lost. We are also working on
implementing other logging targets, which accept longer messages, such as
Fluentd's "forward" input.

### Other

See the `aushape --help` output and experiment!

Contributing
------------

Feel free to open issues, submit pull requests and write to the
[author](mailto:Nikolai.Kondrashov@redhat.com) directly. All contributions are
welcome!
