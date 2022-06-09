+++
title = "Dynamically Add Dock Items with the Jamf Binary"
date = 2022-02-13
path = "jamf-binary-dock"
aliases = [
    "posts/jamf-binary-dynamic-dock/"
]
description = "A quick trick to programmatically add an icon to a user's Dock."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/18"
+++

Adding your organization's common tools or newly-installed items to a user's
Dock can minimize confusion for your colleagues, and is a common task for Mac
admins.

For those managing their fleet with Jamf Pro, the `jamf` binary includes a
`modifyDock` command which allows you to apply certain Dock modifications. It
isn't a fully-featured Dock management tool, but it does include enough
functionality to add new items to a user's Dock.

I was recently working on a project where I needed to conditionally add a Dock
item based on some scripted logic. I wanted to minimize external dependencies,
so I developed a method to leverage the `jamf` binary's built-in Dock management
capability and its `-file` flag to complete the task.

<!-- more -->

First, here's the documentation for the `modifyDock` command, which you can read
by running `sudo jamf help modifyDock`:

```text
Usage:  jamf modifyDock -file <file name> [-leaveRunning] [-beginning] [-remove]
Usage:  jamf modifyDock -id <dock_item_id> [-leaveRunning] [-beginning] [-remove]

  -file          The file that contains the formatted dock items.
  -id            The dock_item_id of the dock item on the JSS.
  -leaveRunning  The Dock process will not be restarted.
  -beginning     The item will be placed at the beginning (left side) of the Dock.
  -remove        The item will be removed instead of added.
```

Typical use of this command assumes use of the `-id` flag, which requires
passing the ID of a "Dock Item" as configured within your Jamf Pro instance.

[Dock Items can be added to Jamf Pro][jamfdockdocs] via the web console or by
using Jamf Admin. Once you've added a Dock Item to Jamf Pro, you can add it to a
Mac's dock by using the Dock Items payload within a Policy, or by calling `sudo
jamf modifyDock -id <Dock Item ID>` on the Mac itself.

However, the `jamf` binary's `modifyDock` command also accepts a `-file` flag.
The built-in help states the flag expects `The file that contains the formatted
dock items.` – a little vague. Online searches didn't turn up much additional
information, and the official Jamf Pro documentation doesn't cover the binary in
great detail.

With some experimentation, I found that the `-file` flag expects an on-disk
path to a file that contains XML representing one or more Dock item entries.

Roughly, the required format of that XML to add an app to the Dock is as
follows:

```xml
<dict>
    <key>GUID</key>
    <integer>-91117049</integer>
    <key>tile-data</key>
    <dict>
        <key>file-data</key>
        <dict>
            <key>_CFURLString</key>
            <string>file://[path to app]/</string>
            <key>_CFURLStringType</key>
            <integer>15</integer>
        </dict>
        <key>file-label</key>
        <string>[item name]</string>
    </dict>
    <key>tile-type</key>
    <string>file-tile</string>
</dict>
```

_Note: The GUID key can be any value, but Jamf Pro uses a consistent value across
all instances, to my knowledge._

That's easy enough to generate the required data on-the-fly within a shell
script, so I wrote a function to do just that.

```bash
dockitem () {
    if [[ -d "${1}" && "${1: -4}" != ".app" ]]; then TYPE="directory"; else TYPE="file"; fi
    TMPFILE=$( /usr/bin/mktemp )
    echo "<dict><key>GUID</key><integer>-91117049</integer><key>tile-data</key><dict><key>file-data</key><dict><key>_CFURLString</key><string>file://${1%%/}/</string><key>_CFURLStringType</key><integer>15</integer></dict><key>file-label</key><string>$( /usr/bin/basename "${1}")</string></dict><key>tile-type</key><string>${TYPE}-tile</string></dict>" > "${TMPFILE}"
    echo "${TMPFILE}"
}
```

This function requires a single argument: the path to the item you want to add
to the Dock.

Stepping through the function, we first determine if the passed path is a
directory that exists on the Mac. If so – and that directory is not actually an
app bundle – the Dock item will be created as a "directory tile," which appears
with a folder icon on the Dock. Otherwise, we assume the path is an app or
individual file.

Next, we generate a temporary file on disk to which we can write the XML
representing the new Dock item. The temporary file's path is captured.

Then we populate a blob of XML with our custom values, and finally print out the
path to the that temporary file.

Using the function is as simple as calling `dockitem` within a script, then
passing a path as an argument.

```bash
dockitem "/Applications/Safari.app"
```

Putting it all together, we can then feed that file to the `jamf` binary's
`modifyDock` command using command substitution like so:

```bash
#!/bin/zsh

dockitem () {
    if [[ -d "${1}" && "${1: -4}" != ".app" ]]; then TYPE="directory"; else TYPE="file"; fi
    TMPFILE=$( /usr/bin/mktemp )
    echo "<dict><key>GUID</key><integer>-91117049</integer><key>tile-data</key><dict><key>file-data</key><dict><key>_CFURLString</key><string>file://${1%%/}/</string><key>_CFURLStringType</key><integer>15</integer></dict><key>file-label</key><string>$( /usr/bin/basename "${1}")</string></dict><key>tile-type</key><string>${TYPE}-tile</string></dict>" > "${TMPFILE}"
    echo "${TMPFILE}"
}

jamf modifyDock -leaveRunning -file $( dockitem "/Applications/Safari.app" )
jamf modifyDock -leaveRunning -file $( dockitem "/Applications/BBEdit.app" )
jamf modifyDock -file $( dockitem "/Applications/Slack.app" )
```

For another example, let's determine the logged-in user's home directory and add
that to the Dock as a directory tile.

```bash
#!/bin/zsh

dockitem () {
    if [[ -d "${1}" && "${1: -4}" != ".app" ]]; then TYPE="directory"; else TYPE="file"; fi
    TMPFILE=$( /usr/bin/mktemp )
    echo "<dict><key>GUID</key><integer>-91117049</integer><key>tile-data</key><dict><key>file-data</key><dict><key>_CFURLString</key><string>file://${1%%/}/</string><key>_CFURLStringType</key><integer>15</integer></dict><key>file-label</key><string>$( /usr/bin/basename "${1}")</string></dict><key>tile-type</key><string>${TYPE}-tile</string></dict>" > "${TMPFILE}"
    echo "${TMPFILE}"
}

CURRENTUSER=$( /usr/sbin/scutil <<< "show State:/Users/ConsoleUser" | /usr/bin/awk '/Name :/ && ! /loginwindow/ { print $3 }' )
HOMEDIR=$( /usr/bin/dscl . read "/Users/${CURRENTUSER}" NFSHomeDirectory )

jamf modifyDock -file $( dockitem "${HOMEDIR}" )
```

This script will read the `NFSHomeDirectory` attribute of the
currently-logged-in user's account and add it to the Dock. Dynamically
determining the user's home directory path is not otherwise possible with a Jamf
Pro policy alone, so this method bridges that gap.

Ultimately, you can probably file this method under "things you should probably
do a better way, but this is also kind of neat and you may find a use."

A purpose-built tool like [docklib][docklib] or [dockutil][dockutil] will offer
wildly more flexibility and control over the Dock.

But... this will work in a pinch for simple needs.

[ejdocklib]: <https://www.elliotjordan.com/posts/resilient-docklib/>
[jamfdockdocs]: <https://docs.jamf.com/jamf-pro/documentation/Dock_Items.html>
[docklib]: <https://github.com/homebysix/docklib/>
[dockutil]: <https://github.com/kcrawford/dockutil>
