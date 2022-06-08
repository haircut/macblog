+++
title = "How to examine the network traffic of MDM enrollment during Setup Assistant"
date = 2022-05-28
description = "Advice on enabling the root account and running tcpdump during macOS Setup Assistant to record network traffic during initial MDM enrollment."
path = "mdmpackettrace"
[extra]
author = "Matthew Warren"
+++

Part of my job is to test (and re-test) first-time setup workflows for new and
repurposed Macs.

I recently needed to analyze the flow of network traffic during initial MDM
enrollment to confirm an on-premise network was permitting all required traffic.

The `tcpdump` tool – included with macOS – is a powerful utility that allows you
to record all network traffic passing through any interface on the Mac. It
requires elevated privileges to run, however. This presents a problem, since we
do not yet have an account capable of running elevated processes during Setup
Assistant. We haven't even created a local account yet!

To work around this, we need to enable the `root` account _before_ proceeding
through **Setup Assistant**.

<!-- more -->

<mark>I strongly recommend doing these sorts of analyses on a dedicated test Mac
that you don't mind erasing.</mark> Gather the data you need, then erase it.

## Reinstall macOS

First, we need to return the Mac to a "fresh" state by reinstalling macOS.

Apple provides [complete instructions on reinstalling the operating system][os].

If you're using a Mac with an Apple Silicon chip, you can _very_ quickly
[restore the Mac using Apple Configurator][configurator].

## Enable `root` from macOS Recovery

Next, we need to _temporarily_ enable the `root` account by setting a
password for it. We'll disable it later, but this is **required** to be able to
run privileged processes during Setup Assistant.

1. Start up the Mac in [Recovery mode][recovery].
2. Once Recovery loads, select *Utilities > Terminal* on the top menu to open a
   Terminal window.
3. Initiate a password reset for the `root` account using the following command,
   depending on whether the Mac has an Apple Silicon or Intel chip:

   _For a Mac with an Apple Silicon chip..._
   
   ```shell
   dscl -f /Volumes/Data/private/var/db/dslocal/nodes/Default localhost -passwd /Local/Default/Users/root
   ```

   _For a Mac with an Intel chip..._

   ```shell
   dscl -f /Volumes/Macintosh\ HD\ -\ Data/private/var/db/dslocal/nodes/Default localhost -passwd /Local/Default/Users/root
   ```
4. When prompted to enter a `New password:`, type in the password you wish to
   use with the `root` account. The value will not be displayed on screen, and
   you will *not* be prompted to confirm it, so use caution.
5. Restart the Mac by typing `reboot` then pressing <kbd>Return</kbd>.

## Open a Terminal during Setup Assistant

When you start up the Mac, you'll see the "hello" screen and Setup Assistant
will begin. Select your language to continue.

Next, press <kbd>⌃ Control</kbd> + <kbd>⌥ Option</kbd> + <kbd>⌘ Command</kbd> +
<kbd>T</kbd> on the keyboard to open a Terminal window.

Terminal will open in the background, and you'll be able to switch back and
forth between the Setup Assistant and Terminal windows.

Setup Assistant runs under the temporary `_mbsetupuser` user account. This is a
standard – rather than administrator – account. Elevate to root by typing `su
-`, then entering the password you previously set for the root account in macOS
Recovery.

{{ figure(src="img/terminal-during-setup-assistant.png") }}

Great, now we have a `root` shell!

## Run `tcpdump`

With a root shell, we can run elevated processes like `tcpdump`.

Advance through Setup Assistant until you reach the **Remote Management**
screen. Switch focus to the Terminal window, then run:

```shell
tcpdump -nn -i any | tee -a /Users/Shared/enrollment.dump
```

This will display all traffic for all network interfaces, and will skip reverse
resolution of network addresses to DNS names. I find these options useful to see
where traffic is flowing, and the unresolved IP addresses and port numbers are
the relevant bits of information I'm after.

I also use the `tee` program to simultaneously print the traffic to standard
output and also save a log to a known location. Writing the output to a file
within `/Users/Shared` ensures the file persists through any reboots and is
accessible once Setup Assistant completes. You could use `tcpdump`'s `-w` flag
to save the output to a file, but this creates a binary file that isn't
immediately readable. Using `tee` is my personal preference.

There are tons of options for `tcpdump`, which is not the point of this post. I
recommend [Apple's developer documentation on recording a packet
trace][appletcpdump], and [Daniel Miessler's tcpdump tutorial][miessler] for
extensive help using the tool.

## Clean up

Once MDM enrollment completes, switch focus to the Terminal window and press
<kbd>⌘ Command</kbd> + <kbd>C</kbd> to quit the `tcmpdump` process.

The packet trace is displayed in the Terminal window for you to analyze. If
you've also `tee`'d the output to a file, you'll be able to copy that
file to another system for analysis after you've completed Setup Assistant.

The only thing left to do is disable `root` login. Do this from your
administrator-level account by running:

```shell
dscl . -create /Users/root UserShell /usr/bin/false
```

Or – since you're doing this on a test device – erase the Mac and start over
fresh!

[os]: <https://support.apple.com/en-us/HT204904>
[configurator]: <https://support.apple.com/guide/apple-configurator-mac/revive-or-restore-a-mac-with-apple-silicon-apdd5f3c75ad/mac>
[recovery]: <https://support.apple.com/guide/mac-help/intro-to-macos-recovery-mchl46d531d6/mac>
[appletcpdump]: <https://developer.apple.com/documentation/network/recording_a_packet_trace>
[miessler]: <https://danielmiessler.com/study/tcpdump/>
