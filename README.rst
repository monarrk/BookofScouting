The Book of Scouting
====================
A book to document hacks, tricks, and discovery for use in FRC.

This book is intended to be concise yet thorough. With that in mind, low level details of each process should be documented, but these details are not necesarily meant to be explained in a human-friendly way. For example, RFCOMM, its usage, and how it's being used will all be discussed, but RFCOMM itself will not be defined.

Table of Contents
-----------------
1. `How Do Scouting`_
#. `Raspberry Pi`_
#. `Bluetooth Hacks`_
#. `How to Not Use JSON`_

.. `How Do Scouting`_
How to Scouting Good
--------------------

**Technical Scouting Process**:
The scouting app and server go through this process:
1. Server begins running
   - Broadcasts RFCOMM signal, typically on port 1
#. Client begins running
   - Locates pi using the pi's MAC address
   - Attempts to connect over RFCOMM
#. Client sends empty data byte string
#. Pi sends team data located at ROOT/teamData.json to client, where ROOT is the full path to the directory that contains teamData.json
#. Pi closes connection
#. Client parses team data and offers team selection
   - Human enters match data
#. Client reconnects to pi
#. Client sends match data to the pi
#. Pi closes connection

.. `Raspberry Pi`_:
**Disable WiFi**:
To disable WiFi, add the following line to ``/boot/config.txt``: ``dtoverlay=pi3-disable-wifi``. An easy command for doing this is ``echo 'dtoverlay=pi3-disable-wifi' >> /boot/config.txt``.

**Bluetooth Packages**:
Bluetooth packages typically used in the past from a system package manager
- bluez-tools
- python3
- python3-pip

Bluetooth packages for python
- pybluez

**Configure systemd**:
The following commands will create and configure systemd services for bluetooth:
        .. code-block::
        cat > /etc/systemd/network/pan0.netdev << EOF
        [NetDev]
        Name=pan0
        Kind=bridge
        EOF

        cat > /etc/systemd/network/pan0.network << EOF
        [Match]
        Name=pan0
        [Network]
        Address=172.20.1.1/24
        DHCPServer=yes
        EOF

        cat > /etc/systemd/system/bt-agent.service << EOF
        [Unit]
        Description=Bluetooth Auth Agent
        [Service]
        ExecStart=/usr/bin/bt-agent -c NoInputNoOutput
        Type=simple
        [Install]
        WantedBy=multi-user.target
        EOF

        cat > /etc/systemd/system/bt-network.service << EOF
        [Unit]
        Description=Bluetooth NEP PAN
        After=pan0.network
        [Service]
        ExecStart=/usr/bin/bt-network -s nap pan-
        Type=simple
        [Install]
        WantedBy=multi-user.target
        EOF

Remeber to enable these services by running ``systemctl enable Foo``, where Foo is the service name.

.. `Bluetooth Hacks`_:
Bluetooth Hacks
---------------

**Fix systemd's bluetooth compatibility mode**:
Systemd, the init system for debian-derived distributions of Linux, defaults to running the Bluetooth Daemon (bluetoothd) without compatibility mode on. If you are trying to use RFCOMM (serial over bluetooth), then your programs will fail. To fix this, you need to edit systemd's configuration file for bluetoothd with the flag ``--compat`` added to the ExecStart key. 

Example configuration file:
        .. code-block::
        [Unit]
        Description=Bluetooth service
        Documentation=man:bluetoothd(8)
        ConditionPathIsDirectory=/sys/class/bluetooth
        [Service]
        Type=dbus
        BusName=org.bluez
        ExecStart=/usr/lib/bluetooth/bluetoothd --compat # Notice the flag here
        NotifyAccess=main
        #WatchdogSec=10
        #Restart=on-failure
        CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
        LimitNPROC=1
        ProtectHome=true
        ProtectSystem=full
        [Install]
        WantedBy=bluetooth.target
        Alias=dbus-org.bluez.service

To test whether or not you have succeeded, you can try running ``sdptool browse local``, assuming ``sdptool`` is installed on your system.

.. `How to Not Use JSON`_:
How to Not Use JSON
===================

**Do not parse JSON as a string**:
Parsing JSON as a string allows for a myriad of errors that could lead to the production of invalid JSON and should be avoided at all costs. Almost every language used by more than 100 people has a library available to parse JSON using types and methods rather than plaintext and you should use them *for sure*.

**Do not do anything before validating your JSON**:
Especially dangerous things, like opening files for writing. If you somehow get invalid JSON and your program blows up, there can be unexpected consequences that you don't know how to fix, which is an unfortunate situation at competitions.
