# RADIUS認証サーバの構築

OS: Ubuntu 16.04, Software: FreeRADIUS, 認証方式：EAP-TTLS/MS-CHAPv2

まず，以下のコマンドでFreeRADIUSと依存・関連するパッケージをインストールする

```
# apt-get install freeradius freeradius-mysql
```

## RADIUS認証サーバの設定

FreeRADIUSの設定ファイルは`/etc/freeradius/`の下にある．

### radiusd.confの編集

まず，RADIUSサーバのベース設定ファイルである`radiusd.conf`を以下のように編集する．

```
# -*- text -*-
##
## radiusd.conf -- FreeRADIUS server configuration file.
##
##      http://www.freeradius.org/
##      $Id$
##

######################################################################
#
#       Read "man radiusd" before editing this file.  See the section
#       titled DEBUGGING.  It outlines a method where you can quickly
#       obtain the configuration you want, without running into
#       trouble.
#
#       Run the server in debugging mode, and READ the output.
#
#               $ radiusd -X
#
#       We cannot emphasize this point strongly enough.  The vast
#       majority of problems can be solved by carefully reading the
#       debugging output, which includes warnings about common issues,
#       and suggestions for how they may be fixed.
#
#       There may be a lot of output, but look carefully for words like:
#       "warning", "error", "reject", or "failure".  The messages there
#       will usually be enough to guide you to a solution.
#
#       If you are going to ask a question on the mailing list, then
#       explain what you are trying to do, and include the output from
#       debugging mode (radiusd -X).  Failure to do so means that all
#       of the responses to your question will be people telling you
#       to "post the output of radiusd -X".

######################################################################
#
#       The location of other config files and logfiles are declared
#       in this file.
#
#       Also general configuration for modules can be done in this
#       file, it is exported through the API to modules that ask for
#       it.
#
#       See "man radiusd.conf" for documentation on the format of this
#       file.  Note that the individual configuration items are NOT
#       documented in that "man" page.  They are only documented here,
#       in the comments.
#
#       As of 2.0.0, FreeRADIUS supports a simple processing language
#       in the "authorize", "authenticate", "accounting", etc. sections.
#       See "man unlang" for details.
#

prefix = /usr
exec_prefix = /usr
sysconfdir = /etc
localstatedir = /var
sbindir = ${exec_prefix}/sbin
logdir = /var/log/freeradius
raddbdir = /etc/freeradius
radacctdir = ${logdir}/radacct

#
#  name of the running server.  See also the "-n" command-line option.
name = freeradius

#  Location of config and logfiles.
confdir = ${raddbdir}
run_dir = ${localstatedir}/run/${name}

# Should likely be ${localstatedir}/lib/radiusd
db_dir = ${raddbdir}

#
# libdir: Where to find the rlm_* modules.
#
#   This should be automatically set at configuration time.
#
#   If the server builds and installs, but fails at execution time
#   with an 'undefined symbol' error, then you can use the libdir
#   directive to work around the problem.
#
#   The cause is usually that a library has been installed on your
#   system in a place where the dynamic linker CANNOT find it.  When
#   executing as root (or another user), your personal environment MAY
#   be set up to allow the dynamic linker to find the library.  When
#   executing as a daemon, FreeRADIUS MAY NOT have the same
#   personalized configuration.
#
#   To work around the problem, find out which library contains that symbol,
#   and add the directory containing that library to the end of 'libdir',
#   with a colon separating the directory names.  NO spaces are allowed.
#
#   e.g. libdir = /usr/local/lib:/opt/package/lib
#
#   You can also try setting the LD_LIBRARY_PATH environment variable
#   in a script which starts the server.
#
#   If that does not work, then you can re-configure and re-build the
#   server to NOT use shared libraries, via:
#
#       ./configure --disable-shared
#       make
#       make install
#
libdir = /usr/lib/freeradius

#  pidfile: Where to place the PID of the RADIUS server.
#
#  The server may be signalled while it's running by using this
#  file.
#
#  This file is written when ONLY running in daemon mode.
#
#  e.g.:  kill -HUP `cat /var/run/radiusd/radiusd.pid`
#
pidfile = ${run_dir}/${name}.pid

#  chroot: directory where the server does "chroot".
#
#  The chroot is done very early in the process of starting the server.
#  After the chroot has been performed it switches to the "user" listed
#  below (which MUST be specified).  If "group" is specified, it switchs
#  to that group, too.  Any other groups listed for the specified "user"
#  in "/etc/group" are also added as part of this process.
#
#  The current working directory (chdir / cd) is left *outside* of the
#  chroot until all of the modules have been initialized.  This allows
#  the "raddb" directory to be left outside of the chroot.  Once the
#  modules have been initialized, it does a "chdir" to ${logdir}.  This
#  means that it should be impossible to break out of the chroot.
#
#  If you are worried about security issues related to this use of chdir,
#  then simply ensure that the "raddb" directory is inside of the chroot,
#  end be sure to do "cd raddb" BEFORE starting the server.
#
#  If the server is statically linked, then the only files that have
#  to exist in the chroot are ${run_dir} and ${logdir}.  If you do the
#  "cd raddb" as discussed above, then the "raddb" directory has to be
#  inside of the chroot directory, too.
#
#chroot = /path/to/chroot/directory

# user/group: The name (or #number) of the user/group to run radiusd as.
#
#   If these are commented out, the server will run as the user/group
#   that started it.  In order to change to a different user/group, you
#   MUST be root ( or have root privleges ) to start the server.
#
#   We STRONGLY recommend that you run the server with as few permissions
#   as possible.  That is, if you're not using shadow passwords, the
#   user and group items below should be set to radius'.
#
#  NOTE that some kernels refuse to setgid(group) when the value of
#  (unsigned)group is above 60000; don't use group nobody on these systems!
#
#  On systems with shadow passwords, you might have to set 'group = shadow'
#  for the server to be able to read the shadow password file.  If you can
#  authenticate users while in debug mode, but not in daemon mode, it may be
#  that the debugging mode server is running as a user that can read the
#  shadow info, and the user listed below can not.
#
#  The server will also try to use "initgroups" to read /etc/groups.
#  It will join all groups where "user" is a member.  This can allow
#  for some finer-grained access controls.
#
user = freerad
group = freerad

#  panic_action: Command to execute if the server dies unexpectedly.
#
#  FOR PRODUCTION SYSTEMS, ACTIONS SHOULD ALWAYS EXIT.
#  AN INTERACTIVE ACTION MEANS THE SERVER IS NOT RESPONDING TO REQUESTS.
#  AN INTERACTICE ACTION MEANS THE SERVER WILL NOT RESTART.
#
#  The panic action is a command which will be executed if the server
#  receives a fatal, non user generated signal, i.e. SIGSEGV, SIGBUS,
#  SIGABRT or SIGFPE.
#
#  This can be used to start an interactive debugging session so
#  that information regarding the current state of the server can
#  be acquired.
#
#  The following string substitutions are available:
#  - %e   The currently executing program e.g. /sbin/radiusd
#  - %p   The PID of the currently executing program e.g. 12345
#
#  Standard ${} substitutions are also allowed.
#
#  An example panic action for opening an interactive session in GDB would be:
#
#panic_action = "gdb %e %p"
#
#  Again, don't use that on a production system.
#
#  An example panic action for opening an automated session in GDB would be:
#
#panic_action = "gdb -silent -x ${raddbdir}/panic.gdb %e %p > ${logdir}/gdb-%e-%p.log 2>&1"
#
#  That command can be used on a production system.
#

#  max_request_time: The maximum time (in seconds) to handle a request.
#
#  Requests which take more time than this to process may be killed, and
#  a REJECT message is returned.
#
#  WARNING: If you notice that requests take a long time to be handled,
#  then this MAY INDICATE a bug in the server, in one of the modules
#  used to handle a request, OR in your local configuration.
#
#  This problem is most often seen when using an SQL database.  If it takes
#  more than a second or two to receive an answer from the SQL database,
#  then it probably means that you haven't indexed the database.  See your
#  SQL server documentation for more information.
#
#  Useful range of values: 5 to 120
#
max_request_time = 30

#  cleanup_delay: The time to wait (in seconds) before cleaning up
#  a reply which was sent to the NAS.
#
#  The RADIUS request is normally cached internally for a short period
#  of time, after the reply is sent to the NAS.  The reply packet may be
#  lost in the network, and the NAS will not see it.  The NAS will then
#  re-send the request, and the server will respond quickly with the
#  cached reply.
#
#  If this value is set too low, then duplicate requests from the NAS
#  MAY NOT be detected, and will instead be handled as seperate requests.
#
#  If this value is set too high, then the server will cache too many
#  requests, and some new requests may get blocked.  (See 'max_requests'.)
#
#  Useful range of values: 2 to 10
#
cleanup_delay = 5

#  max_requests: The maximum number of requests which the server keeps
#  track of.  This should be 256 multiplied by the number of clients.
#  e.g. With 4 clients, this number should be 1024.
#
#  If this number is too low, then when the server becomes busy,
#  it will not respond to any new requests, until the 'cleanup_delay'
#  time has passed, and it has removed the old requests.
#
#  If this number is set too high, then the server will use a bit more
#  memory for no real benefit.
#
#  If you aren't sure what it should be set to, it's better to set it
#  too high than too low.  Setting it to 1000 per client is probably
#  the highest it should be.
#
#  Useful range of values: 256 to infinity
#
max_requests = 1024

#  listen: Make the server listen on a particular IP address, and send
#  replies out from that address. This directive is most useful for
#  hosts with multiple IP addresses on one interface.
#
#  If you want the server to listen on additional addresses, or on
#  additionnal ports, you can use multiple "listen" sections.
#
#  Each section make the server listen for only one type of packet,
#  therefore authentication and accounting have to be configured in
#  different sections.
#
#  The server ignore all "listen" section if you are using '-i' and '-p'
#  on the command line.
#
listen {
        #  Type of packets to listen for.
        #  Allowed values are:
        #       auth    listen for authentication packets
        #       acct    listen for accounting packets
        #       proxy   IP to use for sending proxied packets
        #       detail  Read from the detail file.  For examples, see
        #               raddb/sites-available/copy-acct-to-home-server
        #       status  listen for Status-Server packets.  For examples,
        #               see raddb/sites-available/status
        #       coa     listen for CoA-Request and Disconnect-Request
        #               packets.  For examples, see the file
        #               raddb/sites-available/coa
        #
        type = auth

        #  Note: "type = proxy" lets you control the source IP used for
        #        proxying packets, with some limitations:
        #
        #    * A proxy listener CANNOT be used in a virtual server section.
        #    * You should probably set "port = 0".
        #    * Any "clients" configuration will be ignored.
        #
        #  See also proxy.conf, and the "src_ipaddr" configuration entry
        #  in the sample "home_server" section.  When you specify the
        #  source IP address for packets sent to a home server, the
        #  proxy listeners are automatically created.

        #  IP address on which to listen.
        #  Allowed values are:
        #       dotted quad (1.2.3.4)
        #       hostname    (radius.example.com)
        #       wildcard    (*)
        ipaddr = *

        #  OR, you can use an IPv6 address, but not both
        #  at the same time.
#       ipv6addr = ::   # any.  ::1 == localhost

        #  Port on which to listen.
        #  Allowed values are:
        #       integer port number (1812)
        #       0 means "use /etc/services for the proper port"
        port = 0

        #  Some systems support binding to an interface, in addition
        #  to the IP address.  This feature isn't strictly necessary,
        #  but for sites with many IP addresses on one interface,
        #  it's useful to say "listen on all addresses for eth0".
        #
        #  If your system does not support this feature, you will
        #  get an error if you try to use it.
        #
#       interface = eth0

        #  Per-socket lists of clients.  This is a very useful feature.
        #
        #  The name here is a reference to a section elsewhere in
        #  radiusd.conf, or clients.conf.  Having the name as
        #  a reference allows multiple sockets to use the same
        #  set of clients.
        #
        #  If this configuration is used, then the global list of clients
        #  is IGNORED for this "listen" section.  Take care configuring
        #  this feature, to ensure you don't accidentally disable a
        #  client you need.
        #
        #  See clients.conf for the configuration of "per_socket_clients".
        #
#       clients = per_socket_clients
}

#  This second "listen" section is for listening on the accounting
#  port, too.
#
listen {
        ipaddr = *
#       ipv6addr = ::
        port = 0
        type = acct
#       interface = eth0
#       clients = per_socket_clients
}

#  hostname_lookups: Log the names of clients or just their IP addresses
#  e.g., www.freeradius.org (on) or 206.47.27.232 (off).
#
#  The default is 'off' because it would be overall better for the net
#  if people had to knowingly turn this feature on, since enabling it
#  means that each client request will result in AT LEAST one lookup
#  request to the nameserver.   Enabling hostname_lookups will also
#  mean that your server may stop randomly for 30 seconds from time
#  to time, if the DNS requests take too long.
#
#  Turning hostname lookups off also means that the server won't block
#  for 30 seconds, if it sees an IP address which has no name associated
#  with it.
#
#  allowed values: {no, yes}
#
hostname_lookups = no

#  Core dumps are a bad thing.  This should only be set to 'yes'
#  if you're debugging a problem with the server.
#
#  allowed values: {no, yes}
#
allow_core_dumps = no

#  Regular expressions
#
#  These items are set at configure time.  If they're set to "yes",
#  then setting them to "no" turns off regular expression support.
#
#  If they're set to "no" at configure time, then setting them to "yes"
#  WILL NOT WORK.  It will give you an error.
#
regular_expressions     = yes
extended_expressions    = yes

#
#  Logging section.  The various "log_*" configuration items
#  will eventually be moved here.
#
log {
        #
        #  Destination for log messages.  This can be one of:
        #
        #       files - log to "file", as defined below.
        #       syslog - to syslog (see also the "syslog_facility", below.
        #       stdout - standard output
        #       stderr - standard error.
        #
        #  The command-line option "-X" over-rides this option, and forces
        #  logging to go to stdout.
        #
        destination = files

        #
        #  The logging messages for the server are appended to the
        #  tail of this file if destination == "files"
        #
        #  If the server is running in debugging mode, this file is
        #  NOT used.
        #
        file = ${logdir}/radius.log

        #
        #  If this configuration parameter is set, then log messages for
        #  a *request* go to this file, rather than to radius.log.
        #
        #  i.e. This is a log file per request, once the server has accepted
        #  the request as being from a valid client.  Messages that are
        #  not associated with a request still go to radius.log.
        #
        #  Not all log messages in the server core have been updated to use
        #  this new internal API.  As a result, some messages will still
        #  go to radius.log.  Please submit patches to fix this behavior.
        #
        #  The file name is expanded dynamically.  You should ONLY user
        #  server-side attributes for the filename (e.g. things you control).
        #  Using this feature MAY also slow down the server substantially,
        #  especially if you do thinks like SQL calls as part of the
        #  expansion of the filename.
        #
        #  The name of the log file should use attributes that don't change
        #  over the lifetime of a request, such as User-Name,
        #  Virtual-Server or Packet-Src-IP-Address.  Otherwise, the log
        #  messages will be distributed over multiple files.
        #
        #  Logging can be enabled for an individual request by a special
        #  dynamic expansion macro:  %{debug: 1}, where the debug level
        #  for this request is set to '1' (or 2, 3, etc.).  e.g.
        #
        #       ...
        #       update control {
        #              Tmp-String-0 = "%{debug:1}"
        #       }
        #       ...
        #
        #  The attribute that the value is assigned to is unimportant,
        #  and should be a "throw-away" attribute with no side effects.
        #
        #requests = ${logdir}/radiusd-%{%{Virtual-Server}:-DEFAULT}-%Y%m%d.log
        
        #
        #  Which syslog facility to use, if ${destination} == "syslog"
        #
        #  The exact values permitted here are OS-dependent.  You probably
        #  don't want to change this.
        #
        syslog_facility = daemon

        #  Log the full User-Name attribute, as it was found in the request.
        #
        # allowed values: {no, yes}
        #
        stripped_names = no

        #  Log authentication requests to the log file.
        #
        #  allowed values: {no, yes}
        #
        auth = yes

        #  Log passwords with the authentication requests.
        #  auth_badpass  - logs password if it's rejected
        #  auth_goodpass - logs password if it's correct
        #
        #  allowed values: {no, yes}
        #
        auth_badpass = no
        auth_goodpass = no

        #  Log additional text at the end of the "Login OK" messages.
        #  for these to work, the "auth" and "auth_goopass" or "auth_badpass"
        #  configurations above have to be set to "yes".
        #
        #  The strings below are dynamically expanded, which means that
        #  you can put anything you want in them.  However, note that
        #  this expansion can be slow, and can negatively impact server
        #  performance.
        #
#       msg_goodpass = ""
#       msg_badpass = ""
}

#  The program to execute to do concurrency checks.
checkrad = ${sbindir}/checkrad

# SECURITY CONFIGURATION
#
#  There may be multiple methods of attacking on the server.  This
#  section holds the configuration items which minimize the impact
#  of those attacks
#
security {
        #
        #  max_attributes: The maximum number of attributes
        #  permitted in a RADIUS packet.  Packets which have MORE
        #  than this number of attributes in them will be dropped.
        #
        #  If this number is set too low, then no RADIUS packets
        #  will be accepted.
        #
        #  If this number is set too high, then an attacker may be
        #  able to send a small number of packets which will cause
        #  the server to use all available memory on the machine.
        #
        #  Setting this number to 0 means "allow any number of attributes"
                max_attributes = 200

        #
        #  reject_delay: When sending an Access-Reject, it can be
        #  delayed for a few seconds.  This may help slow down a DoS
        #  attack.  It also helps to slow down people trying to brute-force
        #  crack a users password.
        #
        #  Setting this number to 0 means "send rejects immediately"
        #
        #  If this number is set higher than 'cleanup_delay', then the
        #  rejects will be sent at 'cleanup_delay' time, when the request
        #  is deleted from the internal cache of requests.
        #
        #  Useful ranges: 1 to 5
        reject_delay = 1

        #
        #  status_server: Whether or not the server will respond
        #  to Status-Server requests.
        #
        #  When sent a Status-Server message, the server responds with
        #  an Access-Accept or Accounting-Response packet.
        #
        #  This is mainly useful for administrators who want to "ping"
        #  the server, without adding test users, or creating fake
        #  accounting packets.
        #
        #  It's also useful when a NAS marks a RADIUS server "dead".
        #  The NAS can periodically "ping" the server with a Status-Server
        #  packet.  If the server responds, it must be alive, and the
        #  NAS can start using it for real requests.
        #
        #  See also raddb/sites-available/status
        #
        status_server = no

        #
        #  allow_vulnerable_openssl: Allow the server to start with
        #  versions of OpenSSL known to have critical vulnerabilities.
        #
        #  This check is based on the version number reported by libssl
        #  and may not reflect patches applied to libssl by
        #  distribution maintainers.
        #
        allow_vulnerable_openssl = no
}

# PROXY CONFIGURATION
#
#  proxy_requests: Turns proxying of RADIUS requests on or off.
#
#  The server has proxying turned on by default.  If your system is NOT
#  set up to proxy requests to another server, then you can turn proxying
#  off here.  This will save a small amount of resources on the server.
#
#  If you have proxying turned off, and your configuration files say
#  to proxy a request, then an error message will be logged.
#
#  To disable proxying, change the "yes" to "no", and comment the
#  $INCLUDE line.
#
#  allowed values: {no, yes}
#
proxy_requests  = no
$INCLUDE proxy.conf


# CLIENTS CONFIGURATION
#
#  Client configuration is defined in "clients.conf".
#

#  The 'clients.conf' file contains all of the information from the old
#  'clients' and 'naslist' configuration files.  We recommend that you
#  do NOT use 'client's or 'naslist', although they are still
#  supported.
#
#  Anything listed in 'clients.conf' will take precedence over the
#  information from the old-style configuration files.
#
$INCLUDE clients.conf


# THREAD POOL CONFIGURATION
#
#  The thread pool is a long-lived group of threads which
#  take turns (round-robin) handling any incoming requests.
#
#  You probably want to have a few spare threads around,
#  so that high-load situations can be handled immediately.  If you
#  don't have any spare threads, then the request handling will
#  be delayed while a new thread is created, and added to the pool.
#
#  You probably don't want too many spare threads around,
#  otherwise they'll be sitting there taking up resources, and
#  not doing anything productive.
#
#  The numbers given below should be adequate for most situations.
#
thread pool {
        #  Number of servers to start initially --- should be a reasonable
        #  ballpark figure.
        start_servers = 5

        #  Limit on the total number of servers running.
        #
        #  If this limit is ever reached, clients will be LOCKED OUT, so it
        #  should NOT BE SET TOO LOW.  It is intended mainly as a brake to
        #  keep a runaway server from taking the system with it as it spirals
        #  down...
        #
        #  You may find that the server is regularly reaching the
        #  'max_servers' number of threads, and that increasing
        #  'max_servers' doesn't seem to make much difference.
        #
        #  If this is the case, then the problem is MOST LIKELY that
        #  your back-end databases are taking too long to respond, and
        #  are preventing the server from responding in a timely manner.
        #
        #  The solution is NOT do keep increasing the 'max_servers'
        #  value, but instead to fix the underlying cause of the
        #  problem: slow database, or 'hostname_lookups=yes'.
        #
        #  For more information, see 'max_request_time', above.
        #
        max_servers = 32

        #  Server-pool size regulation.  Rather than making you guess
        #  how many servers you need, FreeRADIUS dynamically adapts to
        #  the load it sees, that is, it tries to maintain enough
        #  servers to handle the current load, plus a few spare
        #  servers to handle transient load spikes.
        #
        #  It does this by periodically checking how many servers are
        #  waiting for a request.  If there are fewer than
        #  min_spare_servers, it creates a new spare.  If there are
        #  more than max_spare_servers, some of the spares die off.
        #  The default values are probably OK for most sites.
        #
        min_spare_servers = 3
        max_spare_servers = 10

        #  When the server receives a packet, it places it onto an
        #  internal queue, where the worker threads (configured above)
        #  pick it up for processing.  The maximum size of that queue
        #  is given here.
        #
        #  When the queue is full, any new packets will be silently
        #  discarded.
        #
        #  The most common cause of the queue being full is that the
        #  server is dependent on a slow database, and it has received
        #  a large "spike" of traffic.  When that happens, there is
        #  very little you can do other than make sure the server
        #  receives less traffic, or make sure that the database can
        #  handle the load.
        #
#       max_queue_size = 65536

        #  There may be memory leaks or resource allocation problems with
        #  the server.  If so, set this value to 300 or so, so that the
        #  resources will be cleaned up periodically.
        #
        #  This should only be necessary if there are serious bugs in the
        #  server which have not yet been fixed.
        #
        #  '0' is a special value meaning 'infinity', or 'the servers never
        #  exit'
        max_requests_per_server = 0
}

# MODULE CONFIGURATION
#
#  The names and configuration of each module is located in this section.
#
#  After the modules are defined here, they may be referred to by name,
#  in other sections of this configuration file.
#
modules {
        #
        #  Each module has a configuration as follows:
        #
        #       name [ instance ] {
        #               config_item = value
        #               ...
        #       }
        #
        #  The 'name' is used to load the 'rlm_name' library
        #  which implements the functionality of the module.
        #
        #  The 'instance' is optional.  To have two different instances
        #  of a module, it first must be referred to by 'name'.
        #  The different copies of the module are then created by
        #  inventing two 'instance' names, e.g. 'instance1' and 'instance2'
        #
        #  The instance names can then be used in later configuration
        #  INSTEAD of the original 'name'.  See the 'radutmp' configuration
        #  for an example.
        #

        #
        #  As of 2.0.5, most of the module configurations are in a
        #  sub-directory.  Files matching the regex /[a-zA-Z0-9_.]+/
        #  are loaded.  The modules are initialized ONLY if they are
        #  referenced in a processing section, such as authorize,
        #  authenticate, accounting, pre/post-proxy, etc.
        #
        $INCLUDE ${confdir}/modules/

        #  Extensible Authentication Protocol
        #
        #  For all EAP related authentications.
        #  Now in another file, because it is very large.
        #
        $INCLUDE eap.conf

        #  Include another file that has the SQL-related configuration.
        #  This is another file only because it tends to be big.
        #
#       $INCLUDE sql.conf

        #
        #  This module is an SQL enabled version of the counter module.
        #
        #  Rather than maintaining seperate (GDBM) databases of
        #  accounting info for each counter, this module uses the data
        #  stored in the raddacct table by the sql modules. This
        #  module NEVER does any database INSERTs or UPDATEs.  It is
        #  totally dependent on the SQL module to process Accounting
        #  packets.
        #
#       $INCLUDE sql/mysql/counter.conf

        #
        #  IP addresses managed in an SQL table.
        #
#       $INCLUDE sqlippool.conf
}

# Instantiation
#
#  This section orders the loading of the modules.  Modules
#  listed here will get loaded BEFORE the later sections like
#  authorize, authenticate, etc. get examined.
#
#  This section is not strictly needed.  When a section like
#  authorize refers to a module, it's automatically loaded and
#  initialized.  However, some modules may not be listed in any
#  of the following sections, so they can be listed here.
#
#  Also, listing modules here ensures that you have control over
#  the order in which they are initalized.  If one module needs
#  something defined by another module, you can list them in order
#  here, and ensure that the configuration will be OK.
#
instantiate {
        #
        #  Allows the execution of external scripts.
        #  The entire command line (and output) must fit into 253 bytes.
        #
        #  e.g. Framed-Pool = `%{exec:/bin/echo foo}`
        exec
        
        #
        #  The expression module doesn't do authorization,
        #  authentication, or accounting.  It only does dynamic
        #  translation, of the form:
        #
        #       Session-Timeout = `%{expr:2 + 3}`
        #
        #  This module needs to be instantiated, but CANNOT be
        #  listed in any other section.  See 'doc/rlm_expr' for
        #  more information.
        #
        #  rlm_expr is also responsible for registering many
        #  other xlat functions such as md5, sha1 and lc.
        #
        #  We do not recommend removing it's listing here.
        expr

        #
        # We add the counter module here so that it registers
        # the check-name attribute before any module which sets
        # it
#       daily
        expiration
        logintime

        # subsections here can be thought of as "virtual" modules.
        #
        # e.g. If you have two redundant SQL servers, and you want to
        # use them in the authorize and accounting sections, you could
        # place a "redundant" block in each section, containing the
        # exact same text.  Or, you could uncomment the following
        # lines, and list "redundant_sql" in the authorize and
        # accounting sections.
        #
        #redundant redundant_sql {
        #       sql1
        #       sql2
        #}
}

######################################################################
#
#       Policies that can be applied in multiple places are listed
#       globally.  That way, they can be defined once, and referred
#       to multiple times.
#
######################################################################
$INCLUDE policy.conf

######################################################################
#
#       Load virtual servers.
#
#       This next $INCLUDE line loads files in the directory that
#       match the regular expression: /[a-zA-Z0-9_.]+/
#
#       It allows you to define new virtual servers simply by placing
#       a file into the raddb/sites-enabled/ directory.
#
$INCLUDE sites-enabled/

######################################################################
#
#       All of the other configuration sections like "authorize {}",
#       "authenticate {}", "accounting {}", have been moved to the
#       the file:
#
#               raddb/sites-available/default
#
#       This is the "default" virtual server that has the same
#       configuration as in version 1.0.x and 1.1.x.  The default
#       installation enables this virtual server.  You should
#       edit it to create policies for your local site.
#
#       For more documentation on virtual servers, see:
#
#               raddb/sites-available/README
#
######################################################################
```

変更した箇所は，`log`セクション内の`auth = yes`の部分と`security`セクション内の`status_server = no`の部分，その外すぐ下にある`proxy_requests = no`の部分．

### clients.confの編集

編集できたら，次はRADIUSクライアント（ルータ，スイッチ，アクセスポイントなど）との認証を管理する設定ファイルである`clients.conf`を以下のように編集する．

```
# -*- text -*-
##
## clients.conf -- client configuration directives
##
##      $Id$

#######################################################################
#
#  Define RADIUS clients (usually a NAS, Access Point, etc.).

#
#  Defines a RADIUS client.
#
#  '127.0.0.1' is another name for 'localhost'.  It is enabled by default,
#  to allow testing of the server after an initial installation.  If you
#  are not going to be permitting RADIUS queries from localhost, we suggest
#  that you delete, or comment out, this entry.
#
#

#
#  Each client has a "short name" that is used to distinguish it from
#  other clients.
#
#  In version 1.x, the string after the word "client" was the IP
#  address of the client.  In 2.0, the IP address is configured via
#  the "ipaddr" or "ipv6addr" fields.  For compatibility, the 1.x
#  format is still accepted.
#
client localhost {
        #  Allowed values are:
        #       dotted quad (1.2.3.4)
        #       hostname    (radius.example.com)
        ipaddr = 127.0.0.1

        #  OR, you can use an IPv6 address, but not both
        #  at the same time.
#       ipv6addr = ::   # any.  ::1 == localhost

        #
        #  A note on DNS:  We STRONGLY recommend using IP addresses
        #  rather than host names.  Using host names means that the
        #  server will do DNS lookups when it starts, making it
        #  dependent on DNS.  i.e. If anything goes wrong with DNS,
        #  the server won't start!
        #
        #  The server also looks up the IP address from DNS once, and
        #  only once, when it starts.  If the DNS record is later
        #  updated, the server WILL NOT see that update.
        #

        #  One client definition can be applied to an entire network.
        #  e.g. 127/8 should be defined with "ipaddr = 127.0.0.0" and
        #  "netmask = 8"
        #
        #  If not specified, the default netmask is 32 (i.e. /32)
        #
        #  We do NOT recommend using anything other than 32.  There
        #  are usually other, better ways to achieve the same goal.
        #  Using netmasks of other than 32 can cause security issues.
        #
        #  You can specify overlapping networks (127/8 and 127.0/16)
        #  In that case, the smallest possible network will be used
        #  as the "best match" for the client.
        #
        #  Clients can also be defined dynamically at run time, based
        #  on any criteria.  e.g. SQL lookups, keying off of NAS-Identifier,
                #  etc.
        #  See raddb/sites-available/dynamic-clients for details.
        #

#       netmask = 32

        #
        #  The shared secret use to "encrypt" and "sign" packets between
        #  the NAS and FreeRADIUS.  You MUST change this secret from the
        #  default, otherwise it's not a secret any more!
        #
        #  The secret can be any string, up to 8k characters in length.
        #
        #  Control codes can be entered vi octal encoding,
        #       e.g. "\101\102" == "AB"
        #  Quotation marks can be entered by escaping them,
        #       e.g. "foo\"bar"
        #
        #  A note on security:  The security of the RADIUS protocol
        #  depends COMPLETELY on this secret!  We recommend using a
        #  shared secret that is composed of:
        #
        #       upper case letters
        #       lower case letters
        #       numbers
        #
        #  And is at LEAST 8 characters long, preferably 16 characters in
        #  length.  The secret MUST be random, and should not be words,
        #  phrase, or anything else that is recognizable.
        #
        #  The default secret below is only for testing, and should
        #  not be used in any real environment.
        #
        secret          = testing123

        #
        #  Old-style clients do not send a Message-Authenticator
        #  in an Access-Request.  RFC 5080 suggests that all clients
        #  SHOULD include it in an Access-Request.  The configuration
        #  item below allows the server to require it.  If a client
        #  is required to include a Message-Authenticator and it does
        #  not, then the packet will be silently discarded.
        #
        #  allowed values: yes, no
        require_message_authenticator = no

        #
        #  The short name is used as an alias for the fully qualified
        #  domain name, or the IP address.
        #
        #  It is accepted for compatibility with 1.x, but it is no
        #  longer necessary in 2.0
        #
#       shortname       = localhost

        #
        # the following three fields are optional, but may be used by
        # checkrad.pl for simultaneous use checks
        #

        #
        # The nastype tells 'checkrad.pl' which NAS-specific method to
        #  use to query the NAS for simultaneous use.
        #
        #  Permitted NAS types are:
        #
                #       cisco
        #       computone
        #       livingston
        #       juniper
        #       max40xx
        #       multitech
        #       netserver
        #       pathras
        #       patton
        #       portslave
        #       tc
        #       usrhiper
        #       other           # for all other types

        #
        nastype     = other     # localhost isn't usually a NAS...

        #
        #  The following two configurations are for future use.
        #  The 'naspasswd' file is currently used to store the NAS
        #  login name and password, which is used by checkrad.pl
        #  when querying the NAS for simultaneous use.
        #
#       login       = !root
#       password    = someadminpas

        #
        #  As of 2.0, clients can also be tied to a virtual server.
        #  This is done by setting the "virtual_server" configuration
        #  item, as in the example below.
        #
#       virtual_server = home1

        #
        #  A pointer to the "home_server_pool" OR a "home_server"
        #  section that contains the CoA configuration for this
        #  client.  For an example of a coa home server or pool,
        #  see raddb/sites-available/originate-coa
#       coa_server = coa
}

# IPv6 Client
#client ::1 {
#       secret          = testing123
#       shortname       = localhost
#}
#
# All IPv6 Site-local clients
#client fe80::/16 {
#       secret          = testing123
#       shortname       = localhost
#}

#client some.host.org {
#       secret          = testing123
#       shortname       = localhost
#}

#
#  You can now specify one secret for a network of clients.
#  When a client request comes in, the BEST match is chosen.
#  i.e. The entry from the smallest possible network.
#
#client 192.168.0.0/24 {
#       secret          = testing123-1
#       shortname       = private-network-1
#}
#
#client 192.168.0.0/16 {
#       secret          = testing123-2
#       shortname       = private-network-2
#}


#client 10.10.10.10 {
#       # secret and password are mapped through the "secrets" file.
#       secret      = testing123
#       shortname   = liv1
#       # the following three fields are optional, but may be used by
#       # checkrad.pl for simultaneous usage checks
#       nastype     = livingston
#       login       = !root
#       password    = someadminpas
#}

client 192.168.2.254 {
        secret = hoge
        shortname = ictsc6-network
}

#######################################################################
#
#  Per-socket client lists.  The configuration entries are exactly
#  the same as above, but they are nested inside of a section.
#
#  You can have as many per-socket client lists as you have "listen"
#  sections, or you can re-use a list among multiple "listen" sections.
#
#  Un-comment this section, and edit a "listen" section to add:
#  "clients = per_socket_clients".  That IP address/port combination
#  will then accept ONLY the clients listed in this section.
#
#clients per_socket_clients {
#       client 192.168.3.4 {
#               secret = testing123
#        }
#}
```

以下のものを追加したのみである．

```
client 192.168.2.254 {
        secret = hoge
        shortname = ictsc6-network
}
```

### usersの編集

次に，ユーザのアカウント情報を管理する`users`を以下のように編集する．

```
#
#       Please read the documentation file ../doc/processing_users_file,
#       or 'man 5 users' (after installing the server) for more information.
#
#       This file contains authentication security and configuration
#       information for each user.  Accounting requests are NOT processed
#       through this file.  Instead, see 'acct_users', in this directory.
#
#       The first field is the user's name and can be up to
#       253 characters in length.  This is followed (on the same line) with
#       the list of authentication requirements for that user.  This can
#       include password, comm server name, comm server port number, protocol
#       type (perhaps set by the "hints" file), and huntgroup name (set by
#       the "huntgroups" file).
#
#       If you are not sure why a particular reply is being sent by the
#       server, then run the server in debugging mode (radiusd -X), and
#       you will see which entries in this file are matched.
#
#       When an authentication request is received from the comm server,
#       these values are tested. Only the first match is used unless the
#       "Fall-Through" variable is set to "Yes".
#
#       A special user named "DEFAULT" matches on all usernames.
#       You can have several DEFAULT entries. All entries are processed
#       in the order they appear in this file. The first entry that
#       matches the login-request will stop processing unless you use
#       the Fall-Through variable.
#
#       If you use the database support to turn this file into a .db or .dbm
#       file, the DEFAULT entries _have_ to be at the end of this file and
#       you can't have multiple entries for one username.
#
#       Indented (with the tab character) lines following the first
#       line indicate the configuration values to be passed back to
#       the comm server to allow the initiation of a user session.
#       This can include things like the PPP configuration values
#       or the host to log the user onto.
#
#       You can include another `users' file with `$INCLUDE users.other'
#

#
#       For a list of RADIUS attributes, and links to their definitions,
#       see:
#
#       http://www.freeradius.org/rfc/attributes.html
#

#
# Deny access for a specific user.  Note that this entry MUST
# be before any other 'Auth-Type' attribute which results in the user
# being authenticated.
#
# Note that there is NO 'Fall-Through' attribute, so the user will not
# be given any additional resources.
#
#lameuser       Auth-Type := Reject
#               Reply-Message = "Your account has been disabled."

#
# Deny access for a group of users.
#
# Note that there is NO 'Fall-Through' attribute, so the user will not
# be given any additional resources.
#
#DEFAULT        Group == "disabled", Auth-Type := Reject
#               Reply-Message = "Your account has been disabled."
#

#
# This is a complete entry for "steve". Note that there is no Fall-Through
# entry so that no DEFAULT entry will be used, and the user will NOT
# get any attributes in addition to the ones listed here.
#
#steve  Cleartext-Password := "testing"
#       Service-Type = Framed-User,
#       Framed-Protocol = PPP,
#       Framed-IP-Address = 172.16.3.33,
#       Framed-IP-Netmask = 255.255.255.0,
#       Framed-Routing = Broadcast-Listen,
#       Framed-Filter-Id = "std.ppp",
#       Framed-MTU = 1500,
#       Framed-Compression = Van-Jacobsen-TCP-IP

#
# This is an entry for a user with a space in their name.
# Note the double quotes surrounding the name.
#
#"John Doe"     Cleartext-Password := "hello"
#               Reply-Message = "Hello, %{User-Name}"

testuser    Cleartext-Password := "fuga"
        	Reply-Message = "Hello, %{User-Name}"

#
# Dial user back and telnet to the default host for that port
#
#Deg    Cleartext-Password := "ge55ged"
#       Service-Type = Callback-Login-User,
#       Login-IP-Host = 0.0.0.0,
#       Callback-Number = "9,5551212",
#       Login-Service = Telnet,
#       Login-TCP-Port = Telnet

#
# Another complete entry. After the user "dialbk" has logged in, the
# connection will be broken and the user will be dialed back after which
# he will get a connection to the host "timeshare1".
#
#dialbk Cleartext-Password := "callme"
#       Service-Type = Callback-Login-User,
#       Login-IP-Host = timeshare1,
#       Login-Service = PortMaster,
#       Callback-Number = "9,1-800-555-1212"

#
# user "swilson" will only get a static IP number if he logs in with
# a framed protocol on a terminal server in Alphen (see the huntgroups file).
#
# Note that by setting "Fall-Through", other attributes will be added from
# the following DEFAULT entries
#
#swilson        Service-Type == Framed-User, Huntgroup-Name == "alphen"
#               Framed-IP-Address = 192.168.1.65,
#               Fall-Through = Yes

#
# If the user logs in as 'username.shell', then authenticate them
# using the default method, give them shell access, and stop processing
# the rest of the file.
#
#DEFAULT        Suffix == ".shell"
#               Service-Type = Login-User,
#               Login-Service = Telnet,
#               Login-IP-Host = your.shell.machine


#
# The rest of this file contains the several DEFAULT entries.
# DEFAULT entries match with all login names.
# Note that DEFAULT entries can also Fall-Through (see first entry).
# A name-value pair from a DEFAULT entry will _NEVER_ override
# an already existing name-value pair.
#

#
# Set up different IP address pools for the terminal servers.
# Note that the "+" behind the IP address means that this is the "base"
# IP address. The Port-Id (S0, S1 etc) will be added to it.
#
#DEFAULT        Service-Type == Framed-User, Huntgroup-Name == "alphen"
#               Framed-IP-Address = 192.168.1.32+,
#               Fall-Through = Yes

#DEFAULT        Service-Type == Framed-User, Huntgroup-Name == "delft"
#               Framed-IP-Address = 192.168.2.32+,
#               Fall-Through = Yes

#
# Sample defaults for all framed connections.
#
#DEFAULT        Service-Type == Framed-User
#       Framed-IP-Address = 255.255.255.254,
#       Framed-MTU = 576,
#       Service-Type = Framed-User,
#       Fall-Through = Yes

#
# Default for PPP: dynamic IP address, PPP mode, VJ-compression.
# NOTE: we do not use Hint = "PPP", since PPP might also be auto-detected
#       by the terminal server in which case there may not be a "P" suffix.
#       The terminal server sends "Framed-Protocol = PPP" for auto PPP.
#
#DEFAULT Framed-Protocol == PPP
#        Framed-Protocol = PPP,
#        Framed-Compression = Van-Jacobson-TCP-IP

#
# Default for CSLIP: dynamic IP address, SLIP mode, VJ-compression.
#
#DEFAULT Hint == "CSLIP"
#        Framed-Protocol = SLIP,
#        Framed-Compression = Van-Jacobson-TCP-IP

#
# Default for SLIP: dynamic IP address, SLIP mode.
#
#DEFAULT Hint == "SLIP"
#        Framed-Protocol = SLIP

#
# Last default: rlogin to our main server.
#
#DEFAULT
#       Service-Type = Login-User,
#       Login-Service = Rlogin,
#       Login-IP-Host = shellbox.ispdomain.com

# #
# # Last default: shell on the local terminal server.
# #
# DEFAULT
#       Service-Type = Administrative-User

# On no match, the user is denied access.
```

有効なのは以下のもののみ．他は全てコメントである．

```
testuser        Cleartext-Password := "fuga"
                Reply-Message = "Hello, %{User-Name}"
```

### eap.confの編集

次に，認証方式を設定する`eap.conf`を以下のように記述する．

```
# -*- text -*-
##
##  eap.conf -- Configuration for EAP types (PEAP, TTLS, etc.)
##
##      $Id$

#######################################################################
#
#  Whatever you do, do NOT set 'Auth-Type := EAP'.  The server
#  is smart enough to figure this out on its own.  The most
#  common side effect of setting 'Auth-Type := EAP' is that the
#  users then cannot use ANY other authentication method.
#
#  EAP types NOT listed here may be supported via the "eap2" module.
#  See experimental.conf for documentation.
#
        eap {
                #  Invoke the default supported EAP type when
                #  EAP-Identity response is received.
                #
                #  The incoming EAP messages DO NOT specify which EAP
                #  type they will be using, so it MUST be set here.
                #
                #  For now, only one default EAP type may be used at a time.
                #
                #  If the EAP-Type attribute is set by another module,
                #  then that EAP type takes precedence over the
                #  default type configured here.
                #
                default_eap_type = ttls

                #  A list is maintained to correlate EAP-Response
                #  packets with EAP-Request packets.  After a
                #  configurable length of time, entries in the list
                #  expire, and are deleted.
                #
                timer_expire     = 60

                #  There are many EAP types, but the server has support
                #  for only a limited subset.  If the server receives
                #  a request for an EAP type it does not support, then
                #  it normally rejects the request.  By setting this
                #  configuration to "yes", you can tell the server to
                #  instead keep processing the request.  Another module
                #  MUST then be configured to proxy the request to
                #  another RADIUS server which supports that EAP type.
                #
                #  If another module is NOT configured to handle the
                #  request, then the request will still end up being
                #  rejected.
                ignore_unknown_eap_types = no

                # Cisco AP1230B firmware 12.2(13)JA1 has a bug.  When given
                # a User-Name attribute in an Access-Accept, it copies one
                # more byte than it should.
                #
                # We can work around it by configurably adding an extra
                # zero byte.
                cisco_accounting_username_bug = no

                #
                #  Help prevent DoS attacks by limiting the number of
                #  sessions that the server is tracking.  For simplicity,
                #  this is taken from the "max_requests" directive in
                #  radiusd.conf.
                max_sessions = ${max_requests}

                # Supported EAP-types

                #
                #  We do NOT recommend using EAP-MD5 authentication
                #  for wireless connections.  It is insecure, and does
                #  not provide for dynamic WEP keys.
                #
                md5 {
                }

                # Cisco LEAP
                #
                #  We do not recommend using LEAP in new deployments.  See:
                #  http://www.securiteam.com/tools/5TP012ACKE.html
                #
                #  Cisco LEAP uses the MS-CHAP algorithm (but not
                #  the MS-CHAP attributes) to perform it's authentication.
                #
                #  As a result, LEAP *requires* access to the plain-text
                #  User-Password, or the NT-Password attributes.
                #  'System' authentication is impossible with LEAP.
                #
                leap {
                }

                #  Generic Token Card.
                #
                #  Currently, this is only permitted inside of EAP-TTLS,
                #  or EAP-PEAP.  The module "challenges" the user with
                #  text, and the response from the user is taken to be
                #  the User-Password.
                #
                #  Proxying the tunneled EAP-GTC session is a bad idea,
                #  the users password will go over the wire in plain-text,
                #  for anyone to see.
                #
                gtc {
                        #  The default challenge, which many clients
                        #  ignore..
                        #challenge = "Password: "

                        #  The plain-text response which comes back
                        #  is put into a User-Password attribute,
                        #  and passed to another module for
                        #  authentication.  This allows the EAP-GTC
                        #  response to be checked against plain-text,
                        #  or crypt'd passwords.
                        #
                        #  If you say "Local" instead of "PAP", then
                        #  the module will look for a User-Password
                        #  configured for the request, and do the
                        #  authentication itself.
                        #
                        auth_type = PAP
                }

                ## EAP-TLS
                #
                #  See raddb/certs/README for additional comments
                #  on certificates.
                #
                #  If OpenSSL was not found at the time the server was
                #  built, the "tls", "ttls", and "peap" sections will
                #  be ignored.
                #
                #  Otherwise, when the server first starts in debugging
                #  mode, test certificates will be created.  See the
                #  "make_cert_command" below for details, and the README
                #  file in raddb/certs
                #
                #  These test certificates SHOULD NOT be used in a normal
                #  deployment.  They are created only to make it easier
                #  to install the server, and to perform some simple
                #  tests with EAP-TLS, TTLS, or PEAP.
                #
                #  See also:
                #
                #  http://www.dslreports.com/forum/remark,9286052~mode=flat
                #
                #  Note that you should NOT use a globally known CA here!
                #  e.g. using a Verisign cert as a "known CA" means that
                #  ANYONE who has a certificate signed by them can
                #  authenticate via EAP-TLS!  This is likely not what you want.
                tls {
                        #
                        #  These is used to simplify later configurations.
                        #
                        certdir = ${confdir}/certs
                        cadir = ${confdir}/certs

                        private_key_password = whatever
                        private_key_file = ${certdir}/server.key

                        #  If Private key & Certificate are located in
                        #  the same file, then private_key_file &
                        #  certificate_file must contain the same file
                        #  name.
                        #
                        #  If CA_file (below) is not used, then the
                        #  certificate_file below MUST include not
                        #  only the server certificate, but ALSO all
                        #  of the CA certificates used to sign the
                        #  server certificate.
                        certificate_file = ${certdir}/server.pem

                        #  Trusted Root CA list
                        #
                        #  ALL of the CA's in this list will be trusted
                        #  to issue client certificates for authentication.
                        #
                        #  In general, you should use self-signed
                        #  certificates for 802.1x (EAP) authentication.
                        #  In that case, this CA file should contain
                        #  *one* CA certificate.
                        #
                        #  This parameter is used only for EAP-TLS,
                        #  when you issue client certificates.  If you do
                        #  not use client certificates, and you do not want
                        #  to permit EAP-TLS authentication, then delete
                        #  this configuration item.
                        CA_file = ${cadir}/ca.pem

                        #
                        #  For DH cipher suites to work, you have to
                        #  run OpenSSL to create the DH file first:
                        #
                        #       openssl dhparam -out certs/dh 1024
                        #
                        dh_file = ${certdir}/dh
                        random_file = /dev/urandom


                        #
                        #  This can never exceed the size of a RADIUS
                        #  packet (4096 bytes), and is preferably half
                        #  that, to accomodate other attributes in
                        #  RADIUS packet.  On most APs the MAX packet
                        #  length is configured between 1500 - 1600
                        #  In these cases, fragment size should be
                        #  1024 or less.
                        #
                #       fragment_size = 1024

                        #  include_length is a flag which is
                        #  by default set to yes If set to
                        #  yes, Total Length of the message is
                        #  included in EVERY packet we send.
                        #  If set to no, Total Length of the
                        #  message is included ONLY in the
                        #  First packet of a fragment series.
                        #
                #       include_length = yes

                        #  Check the Certificate Revocation List
                        #
                        #  1) Copy CA certificates and CRLs to same directory.
                        #  2) Execute 'c_rehash <CA certs&CRLs Directory>'.
                        #    'c_rehash' is OpenSSL's command.
                        #  3) uncomment the lines below.
                        #  5) Restart radiusd
                #       check_crl = yes

                        # Check if intermediate CAs have been revoked.
                #       check_all_crl = yes

                        CA_path = ${cadir}
                       #
                       #  If check_cert_issuer is set, the value will
                       #  be checked against the DN of the issuer in
                       #  the client certificate.  If the values do not
                       #  match, the cerficate verification will fail,
                       #  rejecting the user.
                       #
                       #  In 2.1.10 and later, this check can be done
                       #  more generally by checking the value of the
                       #  TLS-Client-Cert-Issuer attribute.  This check
                       #  can be done via any mechanism you choose.
                       #
                #       check_cert_issuer = "/C=GB/ST=Berkshire/L=Newbury/O=My Company Ltd"

                       #
                       #  If check_cert_cn is set, the value will
                       #  be xlat'ed and checked against the CN
                       #  in the client certificate.  If the values
                       #  do not match, the certificate verification
                       #  will fail rejecting the user.
                       #
                       #  This check is done only if the previous
                       #  "check_cert_issuer" is not set, or if
                       #  the check succeeds.
                       #
                       #  In 2.1.10 and later, this check can be done
                       #  more generally by checking the value of the
                       #  TLS-Client-Cert-CN attribute.  This check
                       #  can be done via any mechanism you choose.
                       #
                #       check_cert_cn = %{User-Name}
                #
                        # Set this option to specify the allowed
                        # TLS cipher suites.  The format is listed
                        # in "man 1 ciphers".
                        cipher_list = "DEFAULT"

                        #
                        # As part of checking a client certificate, the EAP-TLS
                        # sets some attributes such as TLS-Client-Cert-CN. This
                        # virtual server has access to these attributes, and can
                        # be used to accept or reject the request.
                        #
                #       virtual_server = check-eap-tls

                        # This command creates the initial "snake oil"
                        # certificates when the server is run as root,
                        # and via "radiusd -X".
                        #
                        # As of 2.1.11, it *also* checks the server
                        # certificate for validity, including expiration.
                        # This means that radiusd will refuse to start
                        # when the certificate has expired.  The alternative
                        # is to have the 802.1X clients refuse to connect
                        # when they discover the certificate has expired.
                        #
                        # Debugging client issues is hard, so it's better
                        # for the server to print out an error message,
                        # and refuse to start.
                        #
                        make_cert_command = "${certdir}/bootstrap"

                        #
                        #  Elliptical cryptography configuration
                        #
                        #  Only for OpenSSL >= 0.9.8.f
                        #
                        ecdh_curve = "prime256v1"

                        #
                        #  Session resumption / fast reauthentication
                        #  cache.
                        #
                        #  The cache contains the following information:
                        #
                        #  session Id - unique identifier, managed by SSL
                        #  User-Name  - from the Access-Accept
                        #  Stripped-User-Name - from the Access-Request
                        #  Cached-Session-Policy - from the Access-Accept
                        #
                        #  The "Cached-Session-Policy" is the name of a
                        #  policy which should be applied to the cached
                        #  session.  This policy can be used to assign
                        #  VLANs, IP addresses, etc.  It serves as a useful
                        #  way to re-apply the policy from the original
                        #  Access-Accept to the subsequent Access-Accept
                        #  for the cached session.
                        #
                        #  On session resumption, these attributes are
                        #  copied from the cache, and placed into the
                        #  reply list.
                        #
                        #  You probably also want "use_tunneled_reply = yes"
                        #  when using fast session resumption.
                        #
                        cache {
                              #
                              #  Enable it.  The default is "no".
                              #  Deleting the entire "cache" subsection
                              #  Also disables caching.
                              #
                              #  You can disallow resumption for a
                              #  particular user by adding the following
                              #  attribute to the control item list:
                              #
                              #         Allow-Session-Resumption = No
                              #
                              #  If "enable = no" below, you CANNOT
                              #  enable resumption for just one user
                              #  by setting the above attribute to "yes".
                              #
                              enable = no

                              #
                              #  Lifetime of the cached entries, in hours.
                              #  The sessions will be deleted after this
                              #  time.
                              #
                              lifetime = 24 # hours

                              #
                              #  The maximum number of entries in the
                              #  cache.  Set to "0" for "infinite".
                              #
                              #  This could be set to the number of users
                              #  who are logged in... which can be a LOT.
                              #
                              max_entries = 255
                        }

                        #
                        #  As of version 2.1.10, client certificates can be
                        #  validated via an external command.  This allows
                        #  dynamic CRLs or OCSP to be used.
                        #
                        #  This configuration is commented out in the
                        #  default configuration.  Uncomment it, and configure
                        #  the correct paths below to enable it.
                        #
                        verify {
                                #  A temporary directory where the client
                                #  certificates are stored.  This directory
                                #  MUST be owned by the UID of the server,
                                #  and MUST not be accessible by any other
                                #  users.  When the server starts, it will do
                                #  "chmod go-rwx" on the directory, for
                                #  security reasons.  The directory MUST
                                #  exist when the server starts.
                                #
                                #  You should also delete all of the files
                                #  in the directory when the server starts.
                #               tmpdir = /tmp/radiusd

                                #  The command used to verify the client cert.
                                #  We recommend using the OpenSSL command-line
                                #  tool.
                                #
                                #  The ${..CA_path} text is a reference to
                                #  the CA_path variable defined above.
                                #
                                #  The %{TLS-Client-Cert-Filename} is the name
                                #  of the temporary file containing the cert
                                #  in PEM format.  This file is automatically
                                #  deleted by the server when the command
                                #  returns.
                #               client = "/path/to/openssl verify -CApath ${..CA_path} %{TLS-Client-Cert-Filename}"
                        }

                        #
                        #  OCSP Configuration
                        #  Certificates can be verified against an OCSP
                        #  Responder. This makes it possible to immediately
                        #  revoke certificates without the distribution of
                        #  new Certificate Revokation Lists (CRLs).
                        #
                        ocsp {
                              #
                              #  Enable it.  The default is "no".
                              #  Deleting the entire "ocsp" subsection
                              #  Also disables ocsp checking
                              #
                              enable = no

                              #
                              #  The OCSP Responder URL can be automatically
                              #  extracted from the certificate in question.
                              #  To override the OCSP Responder URL set
                              #  "override_cert_url = yes".
                              #
                              override_cert_url = yes

                              #
                              #  If the OCSP Responder address is not
                              #  extracted from the certificate, the
                              #  URL can be defined here.

                              #
                              #  Limitation: Currently the HTTP
                              #  Request is not sending the "Host: "
                              #  information to the web-server.  This
                              #  can be a problem if the OCSP
                              #  Responder is running as a vhost.
                              #
                              url = "http://127.0.0.1/ocsp/"

                              #
                              # If the OCSP Responder can not cope with nonce
                              # in the request, then it can be disabled here.
                              #
                              # For security reasons, disabling this option
                              # is not recommended as nonce protects against
                              # replay attacks.
                              #
                              # Note that Microsoft AD Certificate Services OCSP
                              # Responder does not enable nonce by default. It is
                              # more secure to enable nonce on the responder than
                              # to disable it in the query here.
                              # See http://technet.microsoft.com/en-us/library/cc770413%28WS.10%29.aspx
                              #
                              # use_nonce = yes

                              #
                              # Number of seconds before giving up waiting
                              # for OCSP response. 0 uses system default.
                              #
                              # timeout = 0

                              #
                              # Normally an error in querying the OCSP
                              # responder (no response from server, server did
                              # not understand the request, etc) will result in
                              # a validation failure.
                              #
                              # To treat these errors as 'soft' failures and
                              # still accept the certificate, enable this
                              # option.
                              #
                              # Warning: this may enable clients with revoked
                              # certificates to connect if the OCSP responder
                              # is not available. Use with caution.
                              #
                              # softfail = no
                        }
                }

                #  The TTLS module implements the EAP-TTLS protocol,
                #  which can be described as EAP inside of Diameter,
                #  inside of TLS, inside of EAP, inside of RADIUS...
                #
                #  Surprisingly, it works quite well.
                #
                #  The TTLS module needs the TLS module to be installed
                #  and configured, in order to use the TLS tunnel
                #  inside of the EAP packet.  You will still need to
                #  configure the TLS module, even if you do not want
                #  to deploy EAP-TLS in your network.  Users will not
                #  be able to request EAP-TLS, as it requires them to
                #  have a client certificate.  EAP-TTLS does not
                #  require a client certificate.
                #
                #  You can make TTLS require a client cert by setting
                #
                #       EAP-TLS-Require-Client-Cert = Yes
                #
                #  in the control items for a request.
                #
                ttls {
                        #  The tunneled EAP session needs a default
                        #  EAP type which is separate from the one for
                        #  the non-tunneled EAP module.  Inside of the
                        #  TTLS tunnel, we recommend using EAP-MD5.
                        #  If the request does not contain an EAP
                        #  conversation, then this configuration entry
                        #  is ignored.
                        default_eap_type = mschapv2

                        #  The tunneled authentication request does
                        #  not usually contain useful attributes
                        #  like 'Calling-Station-Id', etc.  These
                        #  attributes are outside of the tunnel,
                        #  and normally unavailable to the tunneled
                        #  authentication request.
                        #
                        #  By setting this configuration entry to
                        #  'yes', any attribute which NOT in the
                        #  tunneled authentication request, but
                        #  which IS available outside of the tunnel,
                        #  is copied to the tunneled request.
                        #
                        # allowed values: {no, yes}
                        copy_request_to_tunnel = no

                        #  The reply attributes sent to the NAS are
                        #  usually based on the name of the user
                        #  'outside' of the tunnel (usually
                        #  'anonymous').  If you want to send the
                        #  reply attributes based on the user name
                        #  inside of the tunnel, then set this
                        #  configuration entry to 'yes', and the reply
                        #  to the NAS will be taken from the reply to
                        #  the tunneled request.
                        #
                        # allowed values: {no, yes}
                        use_tunneled_reply = no

                        #
                        #  The inner tunneled request can be sent
                        #  through a virtual server constructed
                        #  specifically for this purpose.
                        #
                        #  If this entry is commented out, the inner
                        #  tunneled request will be sent through
                        #  the virtual server that processed the
                        #  outer requests.
                        #
                        virtual_server = "inner-tunnel"

                        #  This has the same meaning as the
                        #  same field in the "tls" module, above.
                        #  The default value here is "yes".
                #       include_length = yes
                }

                ##################################################
                #
                #  !!!!! WARNINGS for Windows compatibility  !!!!!
                #
                ##################################################
                #
                #  If you see the server send an Access-Challenge,
                #  and the client never sends another Access-Request,
                #  then
                #
                #               STOP!
                #
                #  The server certificate has to have special OID's
                #  in it, or else the Microsoft clients will silently
                #  fail.  See the "scripts/xpextensions" file for
                #  details, and the following page:
                #
                #       http://support.microsoft.com/kb/814394/en-us
                #
                #  For additional Windows XP SP2 issues, see:
                #
                #       http://support.microsoft.com/kb/885453/en-us
                #
                #
                #  If is still doesn't work, and you're using Samba,
                #  you may be encountering a Samba bug.  See:
                #
                #       https://bugzilla.samba.org/show_bug.cgi?id=6563
                #
                #  Note that we do not necessarily agree with their
                #  explanation... but the fix does appear to work.
                #
                ##################################################

                #
                #  The tunneled EAP session needs a default EAP type
                #  which is separate from the one for the non-tunneled
                #  EAP module.  Inside of the TLS/PEAP tunnel, we
                #  recommend using EAP-MS-CHAPv2.
                #
                #  The PEAP module needs the TLS module to be installed
                #  and configured, in order to use the TLS tunnel
                #  inside of the EAP packet.  You will still need to
                #  configure the TLS module, even if you do not want
                #  to deploy EAP-TLS in your network.  Users will not
                #  be able to request EAP-TLS, as it requires them to
                #  have a client certificate.  EAP-PEAP does not
                #  require a client certificate.
                #
                #
                #  You can make PEAP require a client cert by setting
                #
                #       EAP-TLS-Require-Client-Cert = Yes
                #
                #  in the control items for a request.
                #
                peap {
                        #  The tunneled EAP session needs a default
                        #  EAP type which is separate from the one for
                        #  the non-tunneled EAP module.  Inside of the
                        #  PEAP tunnel, we recommend using MS-CHAPv2,
                        #  as that is the default type supported by
                        #  Windows clients.
                        default_eap_type = mschapv2
                        #  the PEAP module also has these configuration
                        #  items, which are the same as for TTLS.
                        copy_request_to_tunnel = no
                        use_tunneled_reply = no

                        #  When the tunneled session is proxied, the
                        #  home server may not understand EAP-MSCHAP-V2.
                        #  Set this entry to "no" to proxy the tunneled
                        #  EAP-MSCHAP-V2 as normal MSCHAPv2.
                #       proxy_tunneled_request_as_eap = yes

                        #
                        #  The inner tunneled request can be sent
                        #  through a virtual server constructed
                        #  specifically for this purpose.
                        #
                        #  If this entry is commented out, the inner
                        #  tunneled request will be sent through
                        #  the virtual server that processed the
                        #  outer requests.
                        #
                        virtual_server = "inner-tunnel"

                        # This option enables support for MS-SoH
                        # see doc/SoH.txt for more info.
                        # It is disabled by default.
                        #
#                       soh = yes

                        #
                        # The SoH reply will be turned into a request which
                        # can be sent to a specific virtual server:
                        #
#                       soh_virtual_server = "soh-server"
                }

                #
                #  This takes no configuration.
                #
                #  Note that it is the EAP MS-CHAPv2 sub-module, not
                #  the main 'mschap' module.
                #
                #  Note also that in order for this sub-module to work,
                #  the main 'mschap' module MUST ALSO be configured.
                #
                #  This module is the *Microsoft* implementation of MS-CHAPv2
                #  in EAP.  There is another (incompatible) implementation
                #  of MS-CHAPv2 in EAP by Cisco, which FreeRADIUS does not
                #  currently support.
                #
                mschapv2 {
                        #  Prior to version 2.1.11, the module never
                        #  sent the MS-CHAP-Error message to the
                        #  client.  This worked, but it had issues
                        #  when the cached password was wrong.  The
                        #  server *should* send "E=691 R=0" to the
                        #  client, which tells it to prompt the user
                        #  for a new password.
                        #
                        #  The default is to behave as in 2.1.10 and
                        #  earlier, which is known to work.  If you
                        #  set "send_error = yes", then the error
                        #  message will be sent back to the client.
                        #  This *may* help some clients work better,
                        #  but *may* also cause other clients to stop
                        #  working.
                        #
#                       send_error = no
                }
        }
```

`eap`セクション内の`default_eap_type`を`ttls`に変更したのと，`ttls`セクション内の`default_eap_type`を`mschapv2`に変更したのみである．

## RADIUSサーバの動作テスト

以上の設定ができたら，まず以下のコマンドを実行してRADIUSサーバをデバッグモードで起動する．

```
# freeradius -X
```

`Ready to process requests.`といった表示が出ればひとまず起動できている．起動できればコンソールをもう一つ開き，以下のコマンドを実行して認証を試してみる．

```
# radtest testuser fuga localhost 1812 testing123
```

書式は`radtest [ユーザ名] [ユーザパスワード] [RADIUSサーバIP] [RADIUSサーバポート] [クライアントシークレット]`である．

認証に成功すれば`Access-Accept`が返ってくる．

## RADIUSサーバの起動

最後に，RADIUSサーバを起動し，システム起動時にサービスが自動起動するように設定する．

```
# systemctl start freeradius
# systemctl enable freeradius
```

## 参考

- [RADIUSによる認証ネットワーク環境構築のための７ステップ](http://www.virment.com/radius-server-configuration/)
- [802.1x の概要と各種 EAP タイプ](http://www.intel.co.jp/content/www/jp/ja/support/network-and-i-o/wireless-networking/000006999.html#eap)
- [RADIUSによる認証ネットワーク環境構築のための７ステップ](http://www.virment.com/radius-server-configuration/)
- [eduroam.jp - FreeRADIUS設定](http://www.eduroam.jp/docs/conf-freeradius.html)

