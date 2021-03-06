.. _lab-owners-guide:

Lab Owner's Guide
=================

This document collects information that may be useful to all AVE lab owners,
from single workstations to large centralized labs.

Requirements
------------
- `ADB <https://developer.android.com/studio/command-line/adb.html>`_ properly set up

Installation
------------
To install it one must have administrator privileges on the target machine.

.. rubric:: Installation

To install from source code, run::

    make clean
    make install-all

Alternatively, you can make a debian pakage and install it, run::

    make clean
    make debian
    sudo dpkg -i install ave-[ver].deb
    sudo apt-get install -f
    sudo dpkg -i install ave-[ver].deb (try to install again if needed)

Be prepared to enter your login user names, whatever that may be.


Default Configuration Files
+++++++++++++++++++++++++++
The installation creates the file ``/etc/ave/user`` which contains two items:

* **User name**: This tells all AVE services which user account to use to run
  themselves. AVE will not run as ``root``, not even if specified in this file.
* **Home directory**: AVE services and modules will look for their configuration
  files in ``<home directory>/.ave/config``. To not repeat the same explanation
  all the time, AVE documents only refer to this directory as ``.ave/config``.

Some default configuration files are also created under ``.ave/config``. These
control the behavior of AVE services and various equipment classes.

.. Note:: If all you need is to allocate a locally connected handset without
    any supplemental equipment (relays, power meters, etc) then you can stop
    reading now and focus on test job development instead.

Configure Broker
----------------
Out of the box, AVE is configured to only work with locally available equipment
and will not share this equipment with other hosts. This can be improved on by
forwarding allocations to a centralized lab, or by sharing local equipment to a
centralized lab. Which configuration to choose depends on what kind of host is
to be configured.

The forwarding and sharing roles are described in the :ref:`broker system
documentation <broker-network-allocation>`. Please note that a host cannot be
configured for both roles simultanously.

Configure Forwarding
++++++++++++++++++++
This assumes that another host is running a master broker. Two brokers cannot
refer to each other with the forwarding rule. Instructions:

1. Open ``.ave/config/broker.json``
2. Add or edit the "remote" entry, where ``<hostname>`` is the IP address or
   DNS name of the master broker host::

    {
        "remote": {
            "policy": "forward",
            "host"  : "<hostname>",
            "port"  :  4000
        }
    }

3. Save and restart the broker::

    sudo ave-broker --restart

Configure Lab Master
++++++++++++++++++++
This assumes that other hosts will share equipment to this master host. The
master will be configured to require a password from sharing slaves. Note that
this mechanism is only meant to prevent misconfigured slaves from accidental
sharing with the master. It is not a security feature.

1. Open ``.ave/config/authkeys.json``
2. Edit the "share" entry. Any UTF-8 string should be usable but for ease
   of reference you should probably avoid locale specific characters such as
   "ÅÄÖ" and Japanese or Chinese glyphs.
3. Save and restart the broker::

    sudo ave-broker --restart

Configure Sharing
+++++++++++++++++
This assumes that another broker is acting as a master, as described above. To
configure the slave you need the hostname of the master and its share password.

1. Open ``.ave/config/broker.json``
2. Add or edit the "remote" entry, where ``<hostname>`` is the IP address or
   DNS name of the master broker host, and ``<authkey>`` is the share key set
   in ``.ave/config/authkeys.json`` on that host::

    {
        "remote": {
            "policy" : "share",
            "host"   : "<hostname>",
            "port"   :  4000,
            "authkey": "<authkey>"
        }
    }

3. Save and restart the broker::

    sudo ave-broker --restart

Equipment Stacks
++++++++++++++++
The broker cannot automatically figure out what equipment is set up to be used
together with some other equipment. The "togetherness" is known as "stacking".
A full description of the concept and instructions for configuration is given
in the :ref:`broker system documentation <broker-equipment-stacking>`.

The broker needs a restart after a change to the stacking configuration::

    sudo ave-broker --restart


.. Note:: Unique equipment references are made from the ``"uid"`` and ``"type"``
    entries for a single piece (see *Listing Equipment*, below.) There is one
    exception for handsets which, for legacy reasons, are referred to by
    ``"serial"`` instead of ``"uid"``. See the :ref:`broker system documentation
    <broker-equipment-stacking>`.

Configure Supplemental Services
-------------------------------
A number of supplemental services run in the background of an AVE deployment.
To avoid needless repetition the configuration mechanisms of these services are
not detailed here. Please refer to the links given here:

* **Relay server**: Controls relay equipment. A dedicated server is needed to
  split single boards into multiple logical units. Otherwise one would have to
  use a whole board with every single handset, which would be quite wasteful.
  :ref:`Configuration details <relay-config-files>`.
* **Equipment listers**: There is one for every kind of equipment: Handsets,
  relays, power meters and WLAN dongles. A lister's job is to report unique
  equipment profiles to the local broker. To see what the listers add to the
  local broker, please run::

    tail -f /var/tmp/ave-broker.log

  and plug in a piece of equipment.

* **ADB server**: A special wrapper running as ``root`` is used to increase the
  reliability of the ADB server. This is the only AVE component that runs with
  elevated privileges. :ref:`Configuration details <adb-server-config>` and
  :ref:`potential configuration problems <adb-server-behavior>`.

  Sometimes the daemon will exit fairly soon afterwards (within a minute or so).
  In this case simply start it again.

Listing Equipment
-----------------
The broker command line tool can produce simple equipment lists::

    ave-broker --list     # list available equipment
    ave-broker --list-all # list all equipment, allocated or not

This works with both local and shared equipment but is not very sophisticated.
To make a specialized lister, please consult the :ref:`broker API for
administrative clients <broker-admin-api>`.

When to Restart Services
------------------------
The AVE services restart themselves when newer versions are installed. However,
changes to configuration files, limited upgrades of AVE and a few other special
circumstances require manual restarts:

* Restarts after configuration changes have already been covered above, or are
  covered by separate documentation for various supplemental services.
* A broker restart is needed after changing any config in [home]/.ave/config/*.json::

    ave-broker --restart

* ADB server is never restarted automatically, not even by a full AVE release.
  The reason is that doing so would interfere with all currently running test
  jobs. Instead the lab owner has to do this explicitly if a new ADB version is
  shipped by SWD Tools::

    sudo ave-adb-server --restart

* Changes to port numbers in various ``.ave/config`` files cannot be made while
  that service is running. Fully stop the service before editing the config file
  and then start the service again. 
* Changing the "admin" entry in ``.ave/config/authkeys.json`` cannot be done
  while any AVE service is running. Fully stop *all* AVE services before editing
  the "admin" entry, then start them all again.
* Changes to ``/etc/ave/user`` cannot be made while any AVE service is running.
  Fully stop *all* AVE services before editing it, then start them all again.

Debugging Techniques
--------------------
* Internal errors and exceptions of AVE services are logged to various files in
  ``/var/tmp/``.
* It is often instructive to look at AVE's process tree. This set of switches
  shows the actual process names instead of the command line tool that started
  them::

    ps -ejH | grep ave-

  This will e.g. show you all running broker sessions, which may help explain
  why some piece of equipment is considered unavailable.

* If a service has crashed completely it will typically leave a stale PID file
  behind in ``/var/tmp/``. If ``ps`` does not list the service and starting it
  gives an error such as "pid file /var/tmp/ave-broker.pid exists", then you
  can start the service anyway by adding the ``--force`` argument. Never do
  this if the service is in fact running::

    ave-broker --start --force

* If a service is not responding, please run::

    ave-<service> --hickup
    tar -czf hickups.tar.gz .ave/hickup/*
    lsof > lsof.txt
    tar -czf logs.tar.gz /var/tmp/ave-*

  SWD Tools may want to have the archives for analysis.

USB, Power
---------------------
With adequate power supply it should be possible to connect a large number of
handsets to a single host. However, a large number of concurrent flash jobs may
tax the IO capacity of the host so much that client RPC connections start to
time out. The practical limits to concurrent flashing are not known.


One Host, Multiple Users
------------------------
Multiple users may log in on the same host and use a common AVE installation,
but some configuration work is normally required. The configuration files that
were created for the user who made the installation are not only read by the
AVE services, but also by clients that try to connect. The clients will look in
``/etc/ave/user`` to find the ``.ave/config`` directory but will get stuck if
that directory is not readable. The easiest solution is normally that the user
who made the installation also runs::

    chmod -R a+rx ~/.ave
    chmod a+x ~

However, this opens up the home directory of that user a little bit. If this is
not wanted, one may instead change AVE's home directory in ``/etc/ave/user``.
This is, regrettably, more complicated because AVE uses external tools that need
access to additional files from the original user's home directory:

* ``~/.gitconfig``: To be able to use Git related workspace functions.
* ``~/.ssh``: Because the global Git configuration rewrites some host accesses
  to use SSH.

Especially the SSH configuration is tricky to move. If at all possible, avoid
using a non-default home directory until AVE provides better debugging support
for this use case.

Running AVE as Jenkins
----------------------
This is a variation of the *One Host, Multiple Users* scenario.

The host must have a Jenkins user account and a matching home directory with
correct SSH keys, etc. If so:

1. Enter the Jenkins user account name when installing AVE.

If AVE was already installed, consider fully uninstalling it and installing it
again as the Jenkins user.

You may also try this:

1. Stop all AVE services.
2. Edit ``/etc/ave/user``: Change "user" to the Jenkins account and "home" to
   the corresponding home directory.
3. Move the ``.ave`` to the Jenkins home directory.
4. Use ``chmod`` and/or ``chown`` recursively on the moved ``.ave`` directory
   to make it fully owned and controlled by the Jenkins user account.
5. Start all AVE services.
