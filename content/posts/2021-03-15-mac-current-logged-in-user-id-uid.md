+++
title = "Getting the currently logged-in user's ID (UID) for launchctl"
date = 2021-03-15
path = "uid"
aliases = [
    "post/mac-current-logged-in-user-id-uid"
]
description = "A reliable method to determine the logged-in user's user ID (UID) for use with launchctl."
[extra]
author = "Matthew Warren"
+++

Many scripted macOS workflows require determining the username of the currently logged-in user. Whether you wish to execute a command as that user via `su` or you just want to log the username during your script's execution, you may need to query macOS for this information.

This is a solved problem, and Armin Briegel's excellent article on [Getting the current user in macOS](https://scriptingosx.com/2020/02/getting-the-current-user-in-macos-update/) outlines the best method.

In addition to determining the logged-in user's username, you may also need their user ID number, or UID. For example, loading a LaunchAgent as a specific user requires providing that user's UID. 

Previous solutions involved grabbing the logged-in user's username, then feeding that to `id -u $loggedInUser` – but you can get the currently logged-in user's UID in one step.

<!-- more -->

## The code

A slight modification of Armin's recommendation for grabbing the username will give us the UID.

### For bash and zsh

```
loggedInUserID=$( scutil <<< "show State:/Users/ConsoleUser" | awk '/kCGSSessionUserIDKey :/ { print $3 }' )
```

### For shell

Vanilla shell does not support the `<<<` [here-string](https://tldp.org/LDP/abs/html/x17837.html) used in the previous example, so instead we must pipe the `show` statement to `scutil`.

```
loggedInUserID=$( echo "show State:/Users/ConsoleUser" | scutil | awk '/kCGSSessionUserIDKey :/ { print $3 }' )
```

Both variants will return the UID of the currently logged-in user, or an empty string if the Mac is sitting at the login window.

## An example and a note on `launchctl` syntax

With a UID in hand, you can now load a LaunchAgent _as_ that user:

```
launchctl asuser <uid> launchctl load /Library/LaunchAgents/com.org.example.plist
```

However! In macOS Big Sur, I'm finding `launchctl` is more reliable using the launchctl 2.0 syntax (as [documented by Balmes Pavlov](https://babodee.wordpress.com/2016/04/09/launchctl-2-0-syntax/)). This makes sense, as subcommands like `asuser` have been marked as "legacy" for some time now.

Switching from the "legacy" subcommand `asuser` to `bootstrap gui/"${loggedInUserID}"` consistently loads launchd tasks, and is my current recommendation.

A complete example of loading a LaunchAgent as the currently logged-in user:

```
loggedInUserID=$( scutil <<< "show State:/Users/ConsoleUser" | awk '/kCGSSessionUserIDKey :/ && ! /loginwindow/ { print $3 }' )
/bin/launchctl bootstrap gui/"${loggedInUserID}" /Library/LaunchAgents/com.org.example.plist
```
