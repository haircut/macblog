+++
title = "Express Setup, Location Services, Time Zone, and High Sierra"
date = 2018-02-04
updated = 2022-07-06
path = "express-setup-high-sierra"
aliases = [
    "post/express-setup-location-services-time-zone-and-high-sierra",
    "posts/express-setup-location-services-time-zone-and-high-sierra/"
]
description = "Explaining the correct combination of Setup Assistant screens to skip for an optimal user experience."
[extra]
author = "Matthew Warren"
changelog = [
    { date = 2022-07-06, note = "Added note about Express Setup settings being associated to an Apple ID beginning in macOS Monterey." }
]
+++

I recently ran into a snag with our Device Enrollment Program (DEP) workflow.
Users were not being prompted to enable Location Services to automatically set the time zone, nor was the explicit Time Zone selection screen displayed during Setup Assistant.

The result was that devices wound up configured with the default Cupertino, CA
location, and a Pacific time zone.
We're on the East coast – so we'd have to script a change of settings, or worse, have the user manually modify them.

As it turns out, this is an effect of the new "Express Setup" option in macOS High Sierra.

<!-- more -->

## Skipping Setup Assistant screens

{% aside() %}
**Update for macOS Monterey**

As of macOS Monterey, *Express Setup* will only display if the user is logged in to an Apple ID and has previously configured Siri, Location Services, and App Analytics settings while signed in to that Apple ID.
{% end %}

When first booted, if a Mac has network connectivity it contacts Apple to see if it should receive an enrollment configuration from an MDM solution.

In most MDMs you can configure devices to skip any of a handful of Setup Assistant screens.
In Jamf Pro, the configuration looks like this:

{{ figure(src="/img/dep-express-setup/prestage-skip-bad.png") }}

If a screen is skipped, the default options for settings on the screen are configured.
In the case of Location Services, the default option is to **disable** Location Services.

In macOS 10.13.3 and below, running `sudo /usr/libexec/mdmclient dep nag` will output the device's "activation record."

```
Activation record: {
    AllowPairing = 0;
    AnchorCertificates =     (
    );
    AwaitDeviceConfigured = 1;
    ConfigurationURL = "<enrollment url>";
    IsMDMUnremovable = 0;
    IsMandatory = 1;
    IsSupervised = 1;
    OrganizationAddress = "<organization address>";
    OrganizationAddressLine1 = "<organization address>";
    OrganizationCity = <organization city>;
    OrganizationCountry = <organization country>;
    OrganizationDepartment = "<organization department>";
    OrganizationEmail = "<organization email>";
    OrganizationMagic = <organization magic>;
    OrganizationName = "<organization name>";
    OrganizationPhone = "<organization phone>";
    OrganizationSupportPhone = "<organization phone>";
    OrganizationZipCode = <organization zipcode>;
    SkipSetup =     (
        Diagnostics,
        Biometric,
        Restore,
        Registration,
        TOS,
        Payment,
        AppleID,
        iCloudDiagnostics
    );
}
```

{% alert() %}
**Note:** The `mdmclient` command and some terminology are specific to macOS 10.13.3 and below. Future versions of macOS may change the command and/or terminology.
{% end %}

You'll notice the `SkipSetup` dictionary contains the list of Setup Assistant screens to be skipped.

In troubleshooting our issue with Location Services I was perplexed!
The activation record clearly showed that Location Services was _not_ configured to be skipped.
We should be seeing it during Setup Assistant.

A couple of colleagues on the [MacAdmins Slack team](https://macadmins.org) provided me with the necessary clue to resolve the issue. 
Specifically, Nate Van Dam [noted the new behavior](https://macadmins.slack.com/archives/C19MR7EM9/p1511797350000306).

## Express Setup

macOS High Sierra introduces a new "Express Setup" workflow during Setup
Assistant. This screen consolidates the activation of **Siri**, **Location
Services**, and **App Analytics** into one screen.

{{ figure(src="/img/dep-express-setup/express-setup.png") }}

All three of these screens **must not** be skipped in order for Express Setup to
appear. When you click "Continue" on the Express Setup screen, all three
settings will be activated for you.

If you do configure any of these screens to be skipped, Express Setup will not
appear and the Mac may be configured in an undesired way.

The dependency between these three settings is not currently documented in
Apple's [Device Enrollment Program Profile
details](https://developer.apple.com/library/content/documentation/Miscellaneous/Reference/MobileDeviceManagementProtocolRef/4-Profile_Management/ProfileManagement.html#//apple_ref/doc/uid/TP40017387-CH7-SW30).

## Fixing it

Simple – just make sure **Siri**, **Location Services**, and **App Analytics**
are not skipped in your DEP configurations.

{{ figure(src="/img/dep-express-setup/prestage-skip-good.png") }}

Now, when a you run through Setup Assistant the Express Setup screen will
appear. Accepting the default settings will turn Siri, Location Services, and
App Analytics on, and <mark>devices will be configured to use the correct time
zone of their current location.</mark>

## Disabling undesired settings

Since the new Express Setup screen "bundles" these privacy-related options, it
has the side effect of obfuscating your ability to disable them individually.

You can, however, click "Customize Settings" on the Express Setup screen to
invidually disable Siri, Location Services, or App Analytics. It's common to
disable the App Analytics telemetry options, and unfortunately this new workflow
adds several extra clicks accomplish that result.

Hopefully Apple will document this change, and MDM solutions will update their 
interfaces to better clarify these Setup Assistant dependencies.
