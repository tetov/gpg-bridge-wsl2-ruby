# GPG Bridge for WSL1 and WSL2, written in Ruby

This utility forwards requests from gpg clients in WSL1 and WSL2 to
[Gpg4win](https://gpg4win.org/)'s gpg-agent.exe in Windows. It can also
forward ssh requests to gpg-agent.exe, when using a PGP key for ssh
authentication. It is especially useful when you store your PGP key on a
Yubikey, since WSL cannot (yet) access USB devices.

This tool is inspired by
[wsl-gpg-bridge](https://github.com/Riebart/wsl-gpg-bridge), which I have
used in WSL1 together with Gpg4win and a Yubikey.

When WSL2 became available, I found that wsl-gpg-bridge did not work
anymore. The main problem is that WSL2 distributions run in a separate VM,
with a different IP address, and can no longer connect directly with
gpg-agent.exe in Windows (gpg-agent.exe binds to 127.0.0.1).

To solve the connection problem, this solution consists of two bridges (or
proxy applications):

WSL1:

```
gpg/ssh -> (Unix socket) WSL-bridge ->
    -> (TCP socket) Win-bridge -> (TCP/Assuan socket) gpg-agent.exe
```

WSL2:

```
gpg/ssh -> (Unix socket) WSL-bridge -> (Windows Firewall) ->
    -> (TCP socket) Win-bridge -> (TCP/Assuan socket) gpg-agent.exe
```

In WSL2, network traffic to Windows is external, or public. Therefore it
must be allowed by the Windows Firewall.

Since Windows does not support Unix sockets, gpg-agent.exe uses a mechanism
called an Assuan socket. This is a file that contains the TCP port that
gpg-agent is listening on, and a nonce that is sent as authentication after
connecting. The Win-bridge reads the Assuan socket files and connects
directly with gpg-agent.exe (except for the ssh-agent socket).

Gpg4win's gpg-agent.exe does not currently support standard ssh-agent (or
rather, the implementation is broken). Therefore, PuTTY Pageant ssh support
must be enabled in gpg-agent.exe. To communicate with the Pagent server,
the Win-bridge uses [net-ssh](https://github.com/net-ssh/net-ssh).

## Security

Since WSL2 has a different IP address than the Windows host, the Windows
firewall must allow incoming connections to the Win-bridge. Specifically it
must allow incoming connections from 172.16.0.0/12 and 192.168.0.0/16 to
TCP ports 6910-6913 (you can select other ports if you want).

To secure connections with the Win-bridge, a simple nonce authentication
scheme similar to Assuan sockets is used. The Win-bridge stores a nonce in
a file that should only be accessible by the user. By default it saves the
file in the GPG home directory in Windows. The WSL-bridge reads the nonce
from the file and sends it to the Win-bridge to authenticate. This ensures
that only processes that can read the nonce file can authenticate.

## Installation

In Windows, install [Ruby](https://rubyinstaller.org/downloads/). Ensure
that both the ruby and gpg executables are in the Path.

In Windows and WSL, you must install a few ruby gems:

1. In Windows: gem install -N net-ssh sys-proctable
2. In each WSL distribution: gem install -N ptools sys-proctable

Install gpgbridge.rb in a suitable location in the Windows filesystem that
is reachable from both Windows and WSL.

## Usage

In the examples below I have installed gpgbridge.rb in C:\Program1, which
is /mnt/c/Program1 in WSL.

```
$ ruby /mnt/c/Program1/gpgbridge.rb --help
Usage: gpgbridge.rb [options]
    -s, --[no-]enable-ssh-support    Enable proxying of gpg-agent SSH sockets
    -d, --[no-]daemon                Run as a daemon in the background
    -r, --remote-address IPADDR      The remote address of the Windows bridge component. Needed for WSL2. [127.0.0.1]
    -p, --port PORT                  The first port (of three or four) to use for proxying sockets
    -n, --noncefile PATH             The nonce file path (defaults to file in Windows gpg homedir)
    -l, --logfile PATH               The log file path
    -i, --pidfile PATH               The PID file path
    -v, --[no-]verbose               Verbose logging
    -W, --[no-]windows-bridge        Start the Windows bridge (used by the WSL bridge)
    -R, --windows-address IPADDR     The IP address of the Windows bridge. [0.0.0.0]
    -L, --windows-logfile PATH       The log file path of the Windows bridge
    -I, --windows-pidfile PATH       The PID file path of the Windows bridge
    -h, --help                       Prints this help
```

## Example bash helper functions

Copy the [`gpgbridge_helper.sh`](gpgbridge_helper.sh) script to a directory
 accessible from WSL and make it (`chmod +x path/to/gpgbridge_helper.sh`).


Then add `path/to/gpgbridge_helper.sh` to your `~/.bash_profile`,
`~/.bashrc`, `~/.zshrc` or similar. Add `--ssh` to enable SSH forwarding and
`--wsl2` if you are running WSL2 as arguments to `gpgbridge_helper.sh`.

Make sure to edit the paths in the start of the script according to your
setup.

This will start the WSL-bridge in WSL, which will in turn start the
Win-bridge in Windows. Note that only one WSL-bridge will be started per
WSL distribution, and they will all share the same single Win-bridge
running in Windows.

## Known and handled problems

Net/ssh has a hard-coded timeout of 5s when communicating with Pageant.
This does not work well when gpg-agent.exe is the Pageant server, because
the pagent client will probably time out while gpg-agent.exe is prompting
for PIN entry. The result is that ssh fails unless you are really fast with
entering the PIN.

I have currently worked around this by overriding a function in net/ssh to
enable setting a custom timeout. The timeout is now 30s.

## Remote Desktop

When you are using RDP to a remote host, RDP can redirect the local Yubikey
smartcard to the remote host, where the remote gpg-agent.exe can access it.

Sometimes the Yubikey smartcard will be blocked on the local host so that
the remote host cannot access it. When that happens you need to restart
some local services to free the Yubikey for use on the remote host.

I have a Windows batch script that I run as Administrator to handle that
situation. See [rdp_yubikey.cmd](rdp_yubikey.cmd).

Alternatively, removing and re-inserting the Yubikey seems to let RDP grab
the smartcard.
