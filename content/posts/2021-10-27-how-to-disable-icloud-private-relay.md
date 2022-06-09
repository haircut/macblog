+++
title = "How to Disable iCloud Private Relay in macOS Monterey"
date = 2021-10-27
path = "disable-icloud-private-relay"
aliases = [
    "posts/how-to-disable-icloud-private-relay/"
]
description = "Learn how to disable iCloud Private Relay in macOS using a Configuration Profile."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/13"
+++

Apple recently introduced [iCloud Private Relay][about] as an additional
benefit for iCloud+ subscribers. The feature routes Safari web browsing
(and some other insecure Internet traffic) through a semi-anonymizing service
to reduce third parties' ability to profile and track individual users.

However, it may be necessary in some environments to disable iCloud Private
Relay. The feature may interfere with management controls, prevent required
traffic auditing, or complicate troubleshooting procedures.

Apple provides a [guide to prepare your network or service for iCloud Private
Relay][guide], but it's also possible to disable the feature using a
[Restrictions][rest] Configuration Profile.

<!-- more -->

To disable iCloud Private Relay, set the `allowCloudPrivateRelay` key to `false`
in the `com.apple.applicationaccess` domain. An example full Configuration
Profile is below:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>PayloadContent</key>
        <array>
            <dict>
                <key>PayloadDisplayName</key>
                <string>Restrictions</string>
                <key>PayloadIdentifier</key>
                <string>com.apple.applicationaccess.E8C72ECD-7122-4C66-853F-3F3467D1AEF5</string>
                <key>PayloadType</key>
                <string>com.apple.applicationaccess</string>
                <key>PayloadUUID</key>
                <string>1953B7E6-DB5C-4FDA-A579-2EE05978F4B6</string>
                <key>PayloadVersion</key>
                <integer>1</integer>
                <key>allowCloudPrivateRelay</key>
                <false />
            </dict>
        </array>
        <key>PayloadDescription</key>
        <string>Disables the iCloud Private Relay feature.</string>
        <key>PayloadDisplayName</key>
        <string>Disable iCloud Private Relay</string>
        <key>PayloadIdentifier</key>
        <string>E31B0811-3164-49CE-BAA9-67075398DE85</string>
        <key>PayloadOrganization</key>
        <string>Company Name</string>
        <key>PayloadScope</key>
        <string>System</string>
        <key>PayloadType</key>
        <string>Configuration</string>
        <key>PayloadUUID</key>
        <string>ECEB2ECA-B16F-41F8-9909-7DD36FA1609C</string>
        <key>PayloadVersion</key>
        <integer>1</integer>
    </dict>
</plist>
```

This profile is also [available on GitHub][gist].

Once installed, this profile:

1. Stops traffic from routing to `mask.icloud.com` and `mask-h2.icloud.com` at
   the network level.
2. Removes "Private Relay" from the list of services available to enable in
   _System Preferences > Apple ID_.
3. Removes the "Use iCloud Private Relay" checkbox from the "Network" pane in
   System Preferences.

### Requirements

Unlike many Configuration Profiles payloads, the `com.apple.applicationaccess`
payload is re-evaluated after initial installation.

That means this profile can be pre-installed on systems running macOS versions
prior to Monterey. Go ahead and deploy this restriction to your fleet before
they upgrade to macOS Monterey so the configuration takes immediate effect. It
won't have any effect on macOS Big Sur (or previous systems), but will begin
working once the system is upgraded to Monterey.

This restriction *does not* require Supervision.

### Caveat

I've noted a small bug in macOS Monterey 12.0.1. If the Configuration Profile
restricting iCloud Private Relay is installed *while the relay is active*, the
checkbox in _System Preferences > Network_ remains visible.

{{ figure(src="/img/icloud-private-relay-network-preferences.png", alt="iCloud Private Relay checkbox in System Preferences > Network.") }}

Private Relay <mark>is in fact disabled</mark>, and no traffic is routed through
the service. The "Private Relay" feature *is* removed from the listing in
_System Preferences > Apple ID_. This visual bug persists through reboots, but
only occurs when the profile is installed while iCloud Private Relay is already
running.

[about]: <https://support.apple.com/en-us/HT212614>
[guide]: <https://developer.apple.com/support/prepare-your-network-for-icloud-private-relay>
[rest]: <https://developer.apple.com/documentation/devicemanagement/restrictions>
[gist]: <https://gist.github.com/haircut/1dbb4a0ea6988d5bf2c9388b03047faa>
