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
            "text"   : [
                "node=auditdtest.a1959.org type=SYSCALL ...",
                "node=auditdtest.a1959.org type=PROCTITLE ...",
                ...
            ],
            "data"   : {
                "syscall"   : {
                    "syscall"   : ["rt_sigaction","13"],
                    "success"   : ["yes"],
                    "exit"      : ["0"],
                    ...
                },
                "proctitle" : {
                    "proctitle" : ["bash","\"bash\""]
                },
                ...
            }
        },
        ...
    ]


A truncated XML example:

    <?xml version="1.0" encoding="UTF-8"?>
    <log>
        <event serial="194433" time="2016-01-03T02:37:51.394+02:00" host="auditdtest.a1959.org">
            <text>
                <line>node=auditdtest.a1959.org type=SYSCALL ...</line>
                <line>node=auditdtest.a1959.org type=PROCTITLE ...</line>
                ...
            </text>
            <data>
                <syscall>
                    <syscall i="rt_sigaction" r="13"/>
                    <success i="yes"/>
                    <exit i="0"/>
                    ...
                </syscall>
                <proctitle>
                    <proctitle i="bash" r="&quot;bash&quot;"/>
                </proctitle>
                ...
            </data>
        </event>
        ...
    </log>

There is a number of challenges, the main one being both the Linux kernel and
the Auditd code defining record structure and sometimes changing it from
version to version, without an official specification being there. Yet, we
have developed draft schemas for both [JSON](lib/aushape.schema.json) and
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

You will also need to install some autotools.

For RHEL
```
yum -y install autoconf aclocal automake libtool gcc make audit-libs-devel
```

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

If you'd like to also log original audit messages, add `--with-text` option.
If you'd like to limit the logged event message sizes, add
`--max-event-size=SIZE` option, e.g. `--max-event-size=4k` for a four-kilobyte
limit.

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
use rsyslog with default configuration, they would end up in
`/var/log/secure` on Fedora and RHEL, and in `/var/log/auth.log` on
Debian-based systems.

**NOTE**: Some audit events can be large. For example the execve events can be
in the order of megabytes for very long command lines. Most logging servers
will drop long messages silently. Make sure your audit configuration
only logs events which are not too long, limit the maximum logged event size
to have events cropped with the `--max-event-size=SIZE` option, and/or
configure your logging server to accept longer messages.

#### Forwarding to Elasticsearch

Once aushape messages hit the syslog(3) interface, whether it is provided by
journald, or other logging service, they can be forwarded to Elasticsearch for
storage and analysis. Several logging services are available which can do
that, including Logstash, Fluentd, and rsyslog. Since rsyslog is included in
most Linux distros, we'll use it as an example.

First of all, increase the maximum message size rsyslog can handle to be a bit
more than the message sizes you expect to see from aushape. If you decided
that 16kB is enough, then put this before any network setup in rsyslog.conf
(the top of the file is safest):

    $MaxMessageSize 16k

Then load the Elasticsearch output module:

    $ModLoad omelasticsearch

##### Filtering out aushape messages

Before we can feed aushape messages to Elasticsearch we need to strip them of
syslog data to get pure JSON, using a template:

    template(name="aushape" type="list") {
        constant(value="{")
        property(name="msg"
                 regex.expression="{\\(.*\\)"
                 regex.submatch="1")
        constant(value="\n")
    }

Next you'll likely need to filter out aushape messages to put them into a
separate Elasticsearch index. You can set up an action condition to filter by
the logging program name. Aushape logs with `aushape` program name.

However, since any program can log with any program name, that is prone to log
message spoofing. If you'd like to protect against that, you'll need to also
filter by something which is harder to spoof, like the UID of the logging
program. The UID will be zero for aushape running under auditd and audispd.
However filtering needs to be done differently, depending on where rsyslog
receives aushape messages from.

If it serves the rsyslog(3) socket itself, then you'll need to make sure the
corresponding `imuxsock` module has its `Annotate` and `ParseTrusted` options
enabled. E.g. like this:

    module(load="imuxsock" SysSock.Annotate="on" SysSock.ParseTrusted="on")

Then you can use this condition in your filtering action:

    if $!uid == "0" and $programname == "aushape" then {
        # ... actions ...
    }

If rsyslog receives aushape messages from journald, then no extra setup is
needed, and the filtering condition can be this:

    if $!_UID == "0" and $programname == "aushape" then {
        # ... actions ...
    }

Note that the above would only work with rsyslog v8.17.0 and later, due to an
issue preventing it from parsing variable names starting with underscore.

##### Sending the messages

Once your rule condition is established, you can add the actual action sending
aushape messages to Elasticsearch:

    action(name="aushape-elasticsearch"
           type="omelasticsearch"
           server="localhost"
           searchIndex="aushape-rsyslog"
           searchType="aushape"
           bulkmode="on"
           template="aushape")

The action above would send messages formatted with the `aushape` template,
described above, to an Elasticsearch server running on localhost and default
port, and would put them into index `aushape-rsyslog` with type `aushape`,
using the bulk interface.

Add the following action if you want to also send `aushape` messages to a
dedicated file for debugging:

    action(name="aushape-file"
           type="omfile"
           file="/var/log/aushape.log"
           fileCreateMode="0600"
           template="aushape")

Further, if you don't want aushape messages delivered anywhere else you can add
the discard action (`~`) after both of those:

    ~

If you'd like to exclude aushape messages from *any* other logs remember to
put its rule before any other rules in `rsyslog.conf`.

Here is a complete example of a rule matching messages arriving from aushape,
delivered by journald. It sends them to Elasticsearch running on localhost
with default port, puts them into `aushape-rsyslog` index with type `aushape`,
using bulk interface, stores them in `/var/log/aushape.log` file, and then
stops processing, not letting them get anywhere else.

    if $!_UID == "0" and $programname == "aushape" then {
		action(name="aushape-elasticsearch"
			   type="omelasticsearch"
			   server="localhost"
			   searchIndex="aushape-rsyslog"
			   searchType="aushape"
			   bulkmode="on"
			   template="aushape")
		action(name="aushape-file"
			   type="omfile"
			   file="/var/log/aushape.log"
			   fileCreateMode="0600"
			   template="aushape")
		~
	}

### Other

See the `aushape --help` output and experiment!

Contributing
------------

Feel free to open issues, submit pull requests and write to the
[author](mailto:Nikolai.Kondrashov@redhat.com) directly. All contributions are
welcome!
