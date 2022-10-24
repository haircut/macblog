+++
title = "Manage and enforce custom Login and Background items in macOS Ventura"
date = 2022-10-24
description = "How to enable and enforce your custom scripts in macOS Ventura using the new service management Configuration Profile payload."
path = "manage-custom-login-items"
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/24"
+++

macOS Ventura introduces a new feature that provides users an interface to selectively enable and disable Login and Background items.

You may be familiar with "Login Items" – those apps, documents, or server connections you've configured in *System Preferences > Users & Groups > Login Items* to automatically open when you log in.
The feature and interface has remained largely unchanged since Mac OS X Tiger.

macOS Ventura extends this idea to encompass *any* process configured to automatically launch in the background.
`launchd` jobs like LaunchDaemons and LaunchAgents can now be toggled on or off in the new System Settings app.

This means non-apps, like any custom scripts you deploy to your managed Macs, can be easily turned off.
Depending on how you manage your Macs, the ability to trivially disable management processes can have compliance, audit, and user support implications.

Along with the new functionality, Apple is providing a new Configuration Profile payload to manage or "lock on" your organization's login items on MDM-enrolled Macs.
Login and Background items managed by this new payload cannot be disabled by users within the System Settings

This guide will demonstrate how you can sign, package, and "lock on" a custom script.

<!-- more -->

---

Official documentation from Apple is [currently sparse][appledevicemanagement].
Rather than attempting to fully document all aspects of "service management" features, this guide focuses on managing custom shell scripts you might run on your organization's Macs.

First, I'll offer my observations on how managed Login and Background Items work in practice.

Applications configured to run in the background at login appear in *System Settings > General > Login Items* under the *Allow in Background* heading.

{{ figure(src="img/managed-login-items/login-items-apps.png", caption="Two apps configured to run in the background at login.") }}

Things are more complicated with items that are not apps, such as a shell script executed by a `launchd` job.

Running a shell script via a LaunchDaemon or LaunchAgent is a common pattern used by administrators to automatically perform management tasks on managed Macs.

As a minimal example, the following LaunchDaemon plist will run a script at path `/opt/pretendco/example-startup-script.sh` when the Mac starts up:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.pretendco.startup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/pretendco/example-startup-script.zsh</string>
    </array>
</dict>
</plist>
```

If a launchd task executes a script using the `Program` or `ProgramArguments` key, that script's filename appears as the display value in System Settings.

{{ figure(src="img/managed-login-items/login-items-unsigned-script.png", caption="A shell script run by a LaunchDaemon lists that script's filename in System Settings > General > Login Items.") }}

Either of the following `launchd` job configurations results in System Settings listing the item by filename:

```xml
<key>Program</key>
<string>/opt/example.zsh</string>
```

```xml
<key>ProgramArguments</key>
<array>
    <string>/opt/example.zsh</string>
</array>
```

Items are listed according to what they run.
The details of how macOS determines a login item's attribution chain are complex and poorly documented.

The gist is that scripts are attributed by filename in the absence of additional identity information.
I will explain a method to provide that identity information by code signing the script later in this guide.

Before that, let's look at how to enforce Login and Background items via Configuration Profile.

---

## Manage items using MDM

We can use the new `com.apple.servicemanagement` Configuration Profile payload to automatically enable, allow, and "lock on" our `launchd` jobs and associated scripts.

This payload allows you to configure rules to control which Login Items cannot be disabled by a user in _System Settings > General > Login Items_.
**Administrative users can still use `launchctl` to unload a `launchd` job.**
Managing an item via MDM only disables the GUI control to easily toggle an item off.

Rules are defined as an array of dictionaries within a Configuration Profile.
Each rule defines one of five `RuleTypes` that control different items:

| Rule type                | Manages                                                                                                                                                      |
|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `TeamIdentifier`         | Any item signed by that team's certificate(s)                                                                                                                |
| `BundleIdentifier`       | Single item exactly matching a bundle identifier, e.g. `org.macblog.exampleapp`                                                                             |
| `Label`                  | Single `launchd` job definition with exactly matching label, e.g. `org.macblog.startup`                                                                      |
| `BundleIdentifierPrefix` | Any item with a bundle identifier starting with the provided value, e.g. `org.macblog` matches<br>- `org.macblog.example`<br>- `org.macblog.somethingelse`         |
| `LabelPrefix`            | Any `launchd` job definition with a label starting with the provided value, e.g. `org.macblog` matches<br>- `org.macblog.example`<br>- `org.macblog.somethingelse` |

In general, you should use the most explicit rule type applicable to a particular login item.
It may be tempting to use broad `TeamIdentifier` rules for convenience, but if we're getting new controls, I want to use them correctly.
I may not want to manage and "lock on" *any* process a vendor might install, making a `TeamIdentifier` rule inappropriate.

Here's a complete example Configuration Profile to manage two specific LaunchDaemons by label:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>PayloadDisplayName</key>
        <string>Example Login and Background Item Management</string>
        <key>PayloadIdentifier</key>
        <string>com.apple.servicemanagement.CA085574-9E75-4049-AACF-B7546F8B1378.privacy</string>
        <key>PayloadUUID</key>
        <string>E572AD76-1AA0-48CA-872F-3A59784148F4</string>
        <key>PayloadType</key>
        <string>Configuration</string>
        <key>PayloadScope</key>
        <string>System</string>
        <key>PayloadContent</key>
        <array>
            <dict>
                <key>PayloadDescription</key>
                <string>Configures managed Login and Background items.</string>
                <key>PayloadDisplayName</key>
                <string>Example Login and Background Item Management</string>
                <key>PayloadIdentifier</key>
                <string>A2EF9B5B-C13F-432C-8645-8FFE1C2024B7</string>
                <key>PayloadUUID</key>
                <string>023BAA5C-29D4-4CAB-8042-64832AD3E6BB</string>
                <key>PayloadType</key>
                <string>com.apple.servicemanagement</string>
                <key>PayloadOrganization</key>
                <string>Pretendco</string>
                <key>Rules</key>
                <array>
                    <dict>
                        <key>RuleType</key>
                        <string>Label</string>
                        <key>RuleValue</key>
                        <string>com.pretendco.example-startup-script</string>
                        <key>Comment</key>
                        <string>LaunchDaemon that runs a shell script at startup.</string>
                    </dict>
                    <dict>
                        <key>RuleType</key>
                        <string>Label</string>
                        <key>RuleValue</key>
                        <string>com.pretendco.second-example</string>
                        <key>Comment</key>
                        <string>Another, different, example LaunchDaemon.</string>
                    </dict>
                </array>
            </dict>
        </array>
    </dict>
</plist>
```

Apple has provided a new command line tool in Ventura to help discover the values you might need to include in a `com.apple.servicemanagement` profile.
You can run `sudo sfltool dumpbtm` to view a verbose listing of all loaded Login and Background items.

Here, however, I'm focused here on demonstrating how to manage custom background items you create.
Next, I'll walk through preparing a script for deployment.

---

## Packaging a startup script from scratch

Let's step through a complete example of creating a script and LaunchDaemon designed to run at startup.
We'll use [munkipkg][munkipkg] to package these components for deployment. If you're not familiar with munkipkg, [Elliot Jordan's You Might Like MunkiPkg guide][elliot] is an excellent resource.

### Setup

Create a new project and scaffold out the structure with the following commands.

```shell
munkipkg --create macblog-example
mkdir -p macblog-example/payload/Library/LaunchDaemons
mkdir -p macblog-example/payload/opt/macblog
```

The project folder will look like this:

```plaintext
macblog-example
├── build
├── build-info.plist
├── payload
│   ├── Library
│   │   └── LaunchDaemons
│   └── opt
│       └── macblog
└── scripts
```

### The script

For this example, we'll create a simple script that will write a log file when run.

Save the following to `payload/opt/macblog/example-startup-script.zsh`:

```shell
#!/bin/zsh
echo "$( /bin/date ): example script execution" >> /Library/Logs/example.log
exit 0
```

### The LaunchDaemon

Next, create a LaunchDaemon to run the script every time the computer starts up.
There are [many options for launchd job definitions][launchdinfo] but we'll keep it very minimal.

Save the following to `payload/Library/LaunchDaemons/org.macblog.example.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>org.macblog.example</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/macblog/example-startup-script.zsh</string>
    </array>
</dict>
</plist>
```

### The `postinstall` script

In order to set the proper file ownership and permissions, create a `postinstall` script.
This script will run during installation of the package, after the files are in place.

Save the following to `scripts/postinstall` - note the lack of a file extension on this file; you must follow this convention:

```shell
#!/bin/zsh
/bin/chmod 0744 /opt/macblog/example-startup-script.zsh
/usr/sbin/chown root:wheel /opt/macblog/example-startup-script.zsh
/bin/chmod 0644 /Library/LaunchDaemons/org.macblog.example.plist
/usr/sbin/chown root:wheel /Library/LaunchDaemons/org.macblog.example.plist
/bin/launchctl bootstrap system /Library/LaunchDaemons/org.macblog.example.plist
```

This script also loads the LaunchDaemon after installation, so you won't need to reboot the system for the startup script to run for the first time.

### Sign the startup script

I recently wrote a [full guide to signing custom scripts][signing], so I won't detail the entire process again here.
Go read that guide first.

Use a **Developer ID Application** certificate issued by Apple to code sign the startup script:

```shell
/usr/bin/codesign --sign "Developer ID Application: MacBlog.org (SHJS42SFS32S)" \
    --identifier "org.macblog.example" \
    payload/opt/macblog/example-startup-script.zsh
```

By signing the script, we provide identity information to macOS about the authorship of the code.
These details are used to display the attribution of the item in *System Settings > General > Login Items*.

<mark>This is the key to properly managing the login item.
Signing the script provides the idenity information macOS needs to attribute the login item correctly.</mark>

### Edit `build-info.plist`

`build-info.plist` controls various options for how the package is built.
The [MunkiPkg readme details all available build-info keys][munkipkg-keys] and offers guidance on their use.

I suggest you [sign the package][munkipkg-sign] by including `signing_info`.
If we're code signing the deployed startup script, we might as well code sign the deployment package too, right?
You'll need a **Developer ID Installer** certificate from Apple to sign the package.

### Package it all up and test

The full project layout now looks like this:

```plaintext
macblog-example
├── build
├── build-info.plist
├── payload
│   ├── Library
│   │   └── LaunchDaemons
│   │       └── org.macblog.example.plist
│   └── opt
│       └── macblog
│           └── example-startup-script.zsh
└── scripts
    └── postinstall
```

Make sure you're in the directory containing the `macblog-example` project, then run:

```shell
munkipkg macblog-example
```

This outputs your completed package to the `macblog-example/build` directory.

If you install the finalized package, you'll notice `sudo launchctl list` will now report the the `org.macblog.example` job running, and `/Library/Logs/example.log` will show an entry.

More importantly, _System Settings > General > Login Items_ will list a new item attributed to the signing identity you used to sign the script!

{{ figure(src="img/managed-login-items/login-items-signed-script.png", caption="A signed script attributed to the signing identity.") }}

The UI displays the attribution of *what's ultimately being executed* – in this case, the `example-startup-script.zsh` file.
Since that file is signed, the identity information tied to that certificate is shown in *System Settings*.

<mark>**The LaunchDaemon does not determine the attribution shown in the interface.**
Think about it this way: the interface displays identity information about what's running, not what "causes" it to run.</mark>

### Manage the script using MDM

Create a Configuration Profile using the example above.
The specific rule to manage the custom script should look like this:

```xml
<array>
    <dict>
        <key>RuleType</key>
        <string>Label</string>
        <key>RuleValue</key>
        <string>org.macblog.example</string>
        <key>Comment</key>
        <string>Example rule</string>
    </dict>
</array>
```

Note that the `RuleValue` must match the `Label` of the LaunchDaemon we created earlier.

For the sake of completeness, a full example Configuration Profile will look like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>PayloadDisplayName</key>
        <string>Example Login and Background Item Management</string>
        <key>PayloadIdentifier</key>
        <string>com.apple.servicemanagement.CA085574-9E75-4049-AACF-B7546F8B1378.privacy</string>
        <key>PayloadUUID</key>
        <string>E572AD76-1AA0-48CA-872F-3A59784148F4</string>
        <key>PayloadType</key>
        <string>Configuration</string>
        <key>PayloadScope</key>
        <string>System</string>
        <key>PayloadContent</key>
        <array>
            <dict>
                <key>PayloadDescription</key>
                <string>Configures managed Login and Background items.</string>
                <key>PayloadDisplayName</key>
                <string>Example Login and Background Item Management</string>
                <key>PayloadIdentifier</key>
                <string>A2EF9B5B-C13F-432C-8645-8FFE1C2024B7</string>
                <key>PayloadUUID</key>
                <string>023BAA5C-29D4-4CAB-8042-64832AD3E6BB</string>
                <key>PayloadType</key>
                <string>com.apple.servicemanagement</string>
                <key>PayloadOrganization</key>
                <string>Pretendco</string>
                <key>Rules</key>
                <array>
                    <dict>
                        <key>RuleType</key>
                        <string>Label</string>
                        <key>RuleValue</key>
                        <string>org.macblog.example</string>
                        <key>Comment</key>
                        <string>Example rule</string>
                    </dict>
                </array>
            </dict>
        </array>
    </dict>
</plist>
```

When deployed to your endpoints via MDM, the Configuration Profile applies the rules provided.
Since we signed the script being executed, its identity information is shown in Login Items.
The script is flagged as "managed by your organization" and is "locked on."

{{ figure(src="img/managed-login-items/login-items-signed-locked.png", caption="Managed login item locked on, with proper identity information.") }}

All done!

### Avoid pitfalls

I've seen some launchd configurations that use the `ProgramArguments` array to first specify the path to an interpreter, then the path to a script, like so:

```xml
<key>ProgramArguments</key>
<array>
    <string>/bin/zsh</string>
    <string>/opt/example.zsh</string>
</array>
```

This tells macOS that `/bin/zsh` (or bash, or Python, etc) is the process your LaunchDaemon is launching, which your script as an argument.
That's wrong, and it causes macOS to attribute the interpretter as the launched process.

{{ figure(src="img/managed-login-items/login-items-interpretter.png", caption="Listing a script's interpretter in the launchd plist will lead to an incorrect attribution in System Settings.") }}

**Don't do this.**
You should set the path to the script's interpreter in its shebang, then make the script itself executable.

## Future

Managing legacy launchd plists works fine for now in Ventura, but there's a reason [they're marked **legacy**][statusForLegacyURL].

<mark>The concept of "login items" and the new [SMAppService][smappservice] API signals a transition in Apple's recommended application design process.</mark>
All helper resources should live inside an app bundle, and macOS will increasingly expect items to be deployed this way.

In a future version of macOS, it will likely be the **only** method to automatically launch a process.
Deleting an app bundle would delete all associated resources and automated processes.
That's a win for the end user.

With this clear signal toward a future without LaunchDaemons and LaunchAgents, I'm personally going to invest more time in learning Swift.

[updating-smappservice]: <https://developer.apple.com/documentation/servicemanagement/updating_helper_executables_from_earlier_versions_of_macos?language=objc>
[smappservice]: <https://developer.apple.com/documentation/servicemanagement/smappservice?language=objc>
[launchdinfo]: <https://launchd.info>
[munkipkg]: <https://github.com/munki/munki-pkg>
[elliot]: <https://www.elliotjordan.com/posts/munkipkg-01-intro/>
[munkipkg-keys]: <https://github.com/munki/munki-pkg#build-info-keys>
[signing]: <https://macblog.org/sign-scripts/>
[munkipkg-sign]: <https://github.com/munki/munki-pkg#package-signing>
[statusForLegacyURL]: <https://developer.apple.com/documentation/servicemanagement/smappservice/4024717-statusforlegacyurl?language=objc>
[appledevicemanagement]: <https://github.com/apple/device-management/blob/release/mdm/profiles/com.apple.servicemanagement.yaml>
