#  tinc VPN macOS howto

There seem to be few resources about how to set up the VPN software [tinc](https://www.tinc-vpn.org/) under modern macOS systems. I am using macOS 11.6 Big Sur (2021).

If you have any improvements or feedback, please open an issue or pull request!

## Installation

Can be done via Homebrew or Macports. Homebrew command:

    brew install tinc

**Do not** install [tuntaposx](http://tuntaposx.sourceforge.net/)! This driver is no longer supported in recent versions of macOS. It is also not needed, as macOS' native **utun** driver can also be utilized by tinc.

You can then invoke `tincd` via `sudo /usr/local/sbin/tincd`. Use this for key generation and starting `tincd` for testing.

## Config file location

Place your config files in `/usr/local/etc/tinc/` like this:

* `MyNetwork` (use your network name, can be anything)
    * `tinc.conf`
    * `tinc-up`
    * `tinc-down`
    * `hosts`
        * `HostA` (your hostname)
        * `HostB` (remote hostname)
        * *more host files, if desired*

## Config file contents

`MyNetwork`/`tinc.conf`:

    Name = HostA
    DeviceType = utun
    Mode = router
    ConnectTo = HostB

`tinc-up` is where it gets a bit tricky. For some reason, macOS expects all tun interfaces to behave like a point-to-point interface. So we need to specify a remote address instead of a netmask. In order to get a whole subnet / multiple hosts routed through the VPN, I then manually add a static route for the entire subnet. That seems to do the trick for me.

`MyNetwork`/`tinc-up`:

    #!/bin/sh
    ifconfig $INTERFACE up
    ifconfig $INTERFACE 192.168.123.2 192.168.123.1
    route -n add -net 192.168.123.0/24 192.168.123.1

`MyNetwork`/`tinc-down`:

    #!/bin/sh
    ifconfig $INTERFACE down
    route -n del -net 192.168.123.0/24

`MyNetwork`/`hosts`/`HostA`:

    Subnet = 192.168.123.2/32

    -----BEGIN RSA PUBLIC KEY-----
    [...]

`MyNetwork`/`hosts`/`HostA`:

    Subnet = 192.168.123.1/32

    -----BEGIN RSA PUBLIC KEY-----
    [...]

## Set up a launch daemon

Create the file `/Library/LaunchDaemons/tincd.plist`:

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN"
        "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>KeepAlive</key>
        <true/>
        <key>Label</key>
        <string>tincd</string>
        <key>ProgramArguments</key>
        <array>
            <string>/usr/local/sbin/tincd</string>
            <string>-n</string>
            <string>MyNetwork</string>
            <string>-D</string>
        </array>
    </dict>
    </plist>

Then, run:

    sudo launchctl load /Library/LaunchDaemons/tincd.plist
    sudo launchctl start tincd

To stop the daemon:

    sudo launchctl stop tincd

## Further reading

* https://charlesreid1.com/wiki/Tinc
* https://wiki.hamburg.ccc.de/ChaosVPN:MacOSXHowto
* https://www.tinc-vpn.org/examples/osx-install/ (currently broken)
* https://www.tinc-vpn.org/docs/
