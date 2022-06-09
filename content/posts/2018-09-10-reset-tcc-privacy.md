+++
title = "Helping Your Users Reset TCC Privacy Policy Decisions"
date = 2018-09-10
path = "reset-tcc-privacy"
aliases = [
    "post/reset-tcc-privacy",
    "posts/reset-tcc-privacy/"
]
description = "How to set up a Self Service utility to help your users reset their Privacy decisions."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/8"
+++

Taking a cue from iOS, Mac OS X 10.8 "Mountain Lion" introduced new systems to
help users manage access requests to potentially sensitive and private personal
information. When an app required access to a user's Contacts, for instance, a
consent prompt appeared on screen asking the user to allow or disallow this
access.

Broadly, this system is known as `TCC` or _transparency, consent and control_.

With each version of macOS, Apple broadens the scope of privacy controls. The
upcoming release of macOS Mojave expands these controls such that many
previously-permitted interactions will require user consent. Prompts for consent
appear _only once_ when an action first requires approval.

With more to manage and only a single prompt to allow or disallow access, it can
be opaque for users to understand the state of their system. Actions may fail
with no clear indication as to why; a decision to disallow access is easily
forgotten after many weeks or months.

Let's create an easy method to reset these decisions to allow for a fresh start.

<!- -more -->

## Background

Some brief background.

The origins of `TCC` are explained during the "_[How iOS Security Really Works](https://developer.apple.com/videos/play/wwdc2016/705/?time=674)_" session from WWDC 2016.

During WWDC 2018 Apple hosted two sessions explaining the direction of their
privacy and security posture:

- [Your Apps and the Future of macOS Security](https://developer.apple.com/videos/play/wwdc2018/702/)
- [Better Apps through Better Privacy](https://developer.apple.com/videos/play/wwdc2018/718/)

Both sessions are insightful – check them out!

For a deeper dive into changes around Privacy controls in macOS Mojave, I highly
recommend Rich Mogull's excellent article [Mojave’s New Security and Privacy
Protections Face Usability
Challenges](https://tidbits.com/2018/09/10/mojaves-new-security-and-privacy-protections-face-usability-challenges/).

## The Issue

When a user decides to allow or disallow an interaction via a user consent 
prompt, that decision should appear in _System Preferences > Security & Privacy
> Privacy_. In my testing of macOS Mojave, I noticed that some allowed or
disallowed interactions – namely certain "automation" decisions – did not appear
in a user-configurable capacity in _System Preferences_. This effectively makes
affected decisions "permanent" unless a user is comfortable using a command line
utility.

Moreover, there is no GUI-based method to reset all privacy consent decisions
and "start from scratch." Apple does, helpfully, provide a command line utility
to programmatically un-configure consent decisions so that interactions may
prompt for approval again.

## Resetting Privacy Consent Decisions

macOS includes a utility called `tccutil`. This tool allows you to manage the
privacy database and reset decisions you've made regarding TCC-protected
services.

`tccutil` is a simple utility, supporting one command: `reset`

Resetting decisions made for a particular service is straightforward:

```
tccutil reset <service name>
```

And here's a list of known services:

```
Accessibility
AddressBook
All
AppleEvents
Calendar
Camera
Facebook
LinkedIn
Liverpool
Location
MediaLibrary
Microphone
Photos
PhotosAdd
PostEvent
Reminders
ShareKit
SinaWeibo
Siri
SystemPolicyAllFiles
SystemPolicyDeveloperFiles
SystemPolicySysAdminFiles
TencentWeibo
Twitter
Ubiquity
Willow
```

(Thanks to [@opragel](https://github.com/opragel) for extracting the list of
services.)

By way of example, the following command would reset all decisions you've made
regarding access to your calendar events. All apps that access your calendar
would then prompt you again for consent to access the data the next time you
launch them.

```
tccutil reset Calendar
```

In testing this utility I developed a quick script to automate the process of
resetting **all** the above-listed services. You can find [tcc-reset.py on
GitHub](https://gist.github.com/haircut/aeb22c853b0ae4b483a76320ccc8c8e9).

I wanted to provide the same tool to my users as an easy-to-use troubleshooting
utility, so I've expanded it to make it available in Jamf Self Service. This
allows users to reset privacy consent decisions without needing to contact IT.

## Self Service Privacy Consent Reset

The goal here is to provide a mechanism for users to reset all their decisions
to allow or disallow apps to access protected services or files. We'll
accomplish this with a script and a policy.

### The Script

[Self-Service-Reset-Privacy-Consent.py](https://gist.github.com/haircut/4f30c1d5bb3eafbbc74d69f4ada2b378)
is an expansion of my original
[tcc-reset.py](https://gist.github.com/haircut/aeb22c853b0ae4b483a76320ccc8c8e9)
script. Running the script with no parameters will reset all privacy consent
decisions for the currently-logged-in user. Optionally, you may pass the word
`all` as Jamf Parameter 4 to reset privacy consent decisions for all users on a
system.

Add a new Script to your JSS. Give the script a suitable name, then paste in the
contents of
[Self-Service-Reset-Privacy-Consent.py](https://gist.github.com/haircut/4f30c1d5bb3eafbbc74d69f4ada2b378)
in the "Script" tab.

Jump to the "Options" tab. Leave the Priority set to "After", then add a label
for Parameter 4: "Reset all users? (Type 'all')"

{{ figure(src="/img/reset-tcc/reset-tcc-script-parameters.png") }}

Save the Script. Next, we'll set up a policy.

### The Policy

Create a new Policy. Give it an appropriate name; I've chosen "Reset Privacy
Consent Decisions" to be consistent with the language Apple uses to describe TCC
features. Leave all Triggers un-checked; this policy will only be available via
Self Service. I set the Execution Frequency to "Ongoing" to ensure users have
access to the utility whenever they need.

Add your `Self-Service-Reset-Privacy-Consent.py` script to the Scripts payload
of your policy. As mentioned, you can optionally set Parameter 4 to `all` so
that privacy consent is reset for all users on a system. With single-user
systems this shouldn't be necessary. By default the script resets consent
decisions for the currently-logged-in user. However, I've included the option
for those managing labs or other multi-user systems.

{{ figure(src="/img/reset-tcc/policy-script.png") }}

Set an appropriate policy scope in the "Scope" tab. I use the "All Managed
Clients" computer group so the utility is available to all users.

Next, open the "Self Service" tab and check the box to "Make policy available in
Self Service." I set the "Button Name Before Initiation" and "Button Name After
Initiation" to "Reset" so the button always displays the same text.

I check "Ensure that users view the description" and set the following text as
the description:

> Resets your Privacy Consent decisions.
> 
> If you experience application behavior where certain features like Camera, 
> Microphone, or Disk Access do not work as expected, resetting your Privacy 
> Consent decisions will allow applications to prompt you for consent to access
> these features.

Finally, I upload an icon to spruce up the presentation of the policy when
viewed in Self Service. You're welcome to [download the
icon](/resources/reset-tcc/SS-Reset-Privacy-Consent-Decisions.png) for use in
your organization.

{{ figure(src="/img/reset-tcc/SS-Reset-Privacy.png") }}

And there we go! Our users can now reset or "undo" all the decisions they've
made regarding privacy consent. Apps will once again prompt for consent when
initiating a sensitive interaction.

## Wrapping Up

As the release date of macOS Mojave approaches, I'm hopeful Apple will provide
more detailed documentation around these features.  While I applaud their
apparent goal of increased visibility into access of potentially-sensitive user
data, I'm concerned this current implementation will only confuse users who may
not understand the effects of allowing or disallowing certain interactions.

We shouldn't have to engineer scripts to help users manage Privacy features. If
Apple wants users to embrace these features, they must understand them and their
implications on how their Mac functions. Explicit and concise controls provided
in the GUI should display _all_ user decisions and allow for seamless
administration. Resetting these decisions shouldn't be relegated to a command
line tool. Users need clarity. I mean... "transparency" is right there in the
name "Transparency, Consent and Control", right?

For more information, please join the `#tcc` channel on the [MacAdmins Slack
Team](https://macadmins.herokuapp.com/).
