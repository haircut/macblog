+++
title = "Wait, is my Mac Up to Date?"
date = 2018-01-08
updated = 2022-06-05
path = "up-to-date"
aliases = [
    "post/wait-is-my-mac-up-to-date",
    "posts/wait-is-my-mac-up-to-date/"
]
description = "Methods to help your users self serve their macOS update needs."
[extra]
author = "Matthew Warren"
changelog = [
    { date = 2022-06-05, note = "Added advice that these methods are no longer reliable, nor good practice, on modern versions of macOS." }
]
+++

So you've trained your users to use Jamf Pro's Self Service to install
third-party software, but how can we encourage users to self-manage macOS
operating system updates?

Let's create a user-centric, Self Service workflow for checking the status of
available software updates.

<!-- more -->

{% alert(title="Update for 2022") %}
**Do not follow these instructions**. I wrote this post in 2018 when this was
better advice. Changes in recent versions of macOS obviate some of the methods
discussed in the post, and other areas are simply bad or incomplete advice with
the advent of Apple Silicon chips.
{% end %}

## Background

In my shop we support users at all positions of the "technical literacy"
spectrum. Some users may not have familiarity with the Mac App Store or
installing operating system updates. We want to provide a "single place" to
point users for all their computer needs, so we really push Self Service as
_**the**_ place to go for software. We've built a strong "Self Service" culture,
so that often means we need to provide information directly in Self Service.

The most common question we want to answer is "Hey, is my Mac up to date?"

{{ figure(src="/img/wait-is-my-mac-up-to-date/ss-anim.gif") }}

With a handful of policies, Smart Groups, and some clever scoping, we can help
users answer this question.

## Collecting Available Software Updates

For this workflow, we need to enable the Jamf Pro option to collect available
Software Updates during recon on our clients. This option is found at
_Management Settings > Inventory Collection_; place a check by "Collect
available software updates".

{{ figure(src="/img/wait-is-my-mac-up-to-date/inventory-collection.png") }}

A colleague at another organization cautioned me that collecting available
software updates can lead to database bloat. In my case, we have 500 clients and
the associated `available_software_updates` table in our Jamf database is only
64 KB. In my experience, this workflow encourages users to install available
updates, so the database never has to track a large amount of pending updates
across our fleet. 

You mileage will vary depending on your environment so make sure you monitor
your database!

## Software Update Smart Groups

Once you've enabled collection of available Software Updates, new criteria are
available for your Smart Groups. Here, we're concerned with the "Number of
Available Software Updates" criterion.

Create a new Smart Group named "Software Update - Updates Available" with the
single criterion "Number of Available Updates **more than** `0`".

{{ figure(src="/img/wait-is-my-mac-up-to-date/sg-updates-available.png") }}

Any client with any number of Software Updates available will fall into this
Smart Group.

Next, create another new Smart Group named "Software Update - No Updates
Available". This will be – as you might guess – the inverse of the "Software
Update - No Updates Available" group you just created. Give it the single
criterion "Number of Available Updates **is** `0`".

{{ figure(src="/img/wait-is-my-mac-up-to-date/sg-no-updates-available.png") }}

Clients that are completely up-to-date will fall into this Smart Group.

## Self Service Policies

Now we'll build the two policies a user might see in Self Service.

### "No macOS Updates Available" Policy

First, we'll create a very simple policy to communicate to your users that no
macOS Software Updates are available. This policy is informational only and is,
more or less, a "dummy" policy. We're just giving users a simple and definitive
confirmation their Mac is up to date.

Create a new policy named "No macOS Updates Available". We set the Execution
Frequency to "Ongoing" because this policy will be situationally displayed
multiple times over the life of a device. Do not configure any Triggers since
this policy will only ever be available in Self Service.

{{ figure(src="/img/wait-is-my-mac-up-to-date/p-no-updates-ss-config.png") }}

Next, set the scope of the policy to your "Software Update - No Updates
Available" Smart Group.

Under the Self Service options, ensure "Make the policy available in Self
Service" is checked. Set the "Self Service Display Name" to "No macOS Updates
Available". Both the "Button Name Before Initiation" and "Button Name After
Initiation" should be set, simply, to "Info".

Spend some time crafting an appropriate message for the Self Service
Description. We need to indicate to the user that their Mac is updated and there
is no need to do anything further. Here's an approximation of what I'm using.

> Your computer is up to date, and no **macOS updates** are available or required.
>
> **There is no need to click the "Info" button – this message is purely informational.**
>
> Third-party applications may still require updates. You can check for non-operating-system updates under the "Available Updates" category in Self Service, or by running an application's "Check for Updates" function.
> 
> If you feel this message is in error or you require additional assistance, please contact the helpdesk at &lt;phone number&gt; or &lt;email address&gt;.

Make sure you enable the "Ensure that users view the description" option. If
your users do click the "Info" button for this policy, we want them to see our
helpful description.

{{ figure(src="/img/wait-is-my-mac-up-to-date/ss-no-updates-description.png") }}

Upload a suitable icon for the policy. I've provided the icons shown here as
free-as-in-beer images under the "Resources" heading of this article, but feel
free to use your own.

I check the box to "Include the policy in the Featured category". This puts the
policy front-and-center when a user opens Self Service.

Save the policy. 

Finally, users **will** try to click "Info" again and run the policy no matter
how clear your messaging is. I have the policy logs to prove it! We'll add a
simple feedback script to provide a bit of clarity.

Add the following Script to your Jamf Pro Server:

```shell
#!/bin/bash
/usr/bin/osascript -e 'tell application "Self Service" to display dialog "No macOS updates available." buttons {"Okay"} with icon caution giving up after 120'
```

In case a user clicks "Info" to run the policy – and they will! – they'll
receive a quick secondary confirmation dialog to let them know that no updates
are available.

{{ figure(src="/img/wait-is-my-mac-up-to-date/dialog-no-updates.png") }}

Go back to your "No macOS Updates Available" policy and add this script.

### "Install macOS Updates" Policy

Now we'll handle devices that _do_ have updates available. Create a new policy
named "Install macOS Updates". This policy will also be configured with an
"Ongoing" Execution Frequency, but no Triggers.

Configure the "Software Updates" policy payload to install updates from "Each
computer's default software update server" or "Apple's Software Update server"
as appropriate for your environment. Unless you run a local Software Update
Server, the "Apple's Software Update server" is fine.

{{ figure(src="/img/wait-is-my-mac-up-to-date/p-updates-swu.png") }}

Since some updates will require rebooting the computer, configure the "Restart
Options" policy payload. Set the Startup Disk to "Current Startup Disk (No
Bless)", the No User Logged In Action to "Restart if a package or updates
requires it", and the User Logged In Action to "Restart if a package or updates
requires it". Leave restart Delay as the default `5` minutes, or set a value of
your preference.

{{ figure(src="/img/wait-is-my-mac-up-to-date/p-updates-restart-options-nb.png") }}

With these options, if none of the available Software Updates require a restart
they will simply be installed. But, if one or more updates do require a restart,
the policy will take care of it.

<article class="message is-warning">
	<div class="message-body">
	  <strong>Update 2018-01-15:</strong> 
	  <p>A previous version of this article advised using the "Current Startup Disk" setting within the Restart Options payload.</p>
	  <p>Shortly after this article was published, Apple released <a href="https://support.apple.com/en-us/HT208397">macOS High Sierra 10.13.2 Supplemental Update</a>. In testing, the <code><a href="https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man8/bless.8.html">bless</a></code> issued by Jamf can cause a Mac to restart to the incorrect volume after installing the supplemental update.</p>
	  <p>Selecting the "Current Startup Disk (No Bless)" option avoids this issue.</p>
	</div>
</article>

Configure the "Maintenance" policy payload, and ensure "Update Inventory" is
checked. This will make sure that, once the available updates are installed, a
new inventory report is collected for the device. This will update the inventory
record to show that no updates are required. 

Set the Scope of the policy to your "Software Update - Updates Available" Smart
Group.

Under the Self Service policy options, ensure "Make the policy available in Self
Service" is checked.

Set the Self Service Display Name to "Install macOS Updates", and both the
"Button Name Before Initiation" and "Button Name After Initiation" to "Update".

Provide a helpful description, and check "Ensure that users view the
description". My verbiage is:

> Installs all available Apple software updates for macOS.
>
> **Warning**: You must close all applications to install this system update, and will be required to restart the computer to complete installation. Please save your work before installing updates.

{{ figure(src="/img/wait-is-my-mac-up-to-date/p-updates-ss-config.png") }}

Finally, we'll add some messaging to guide users through the process. Under the
User Interaction section of the policy, configure a suitable "Start Message";
something simple to let the user know the process has begun.

> macOS Software Updates are downloading now and will be installed soon. Please
> save your work.

Set a Restart Message to inform users their Mac must restart if required by an
update. This message will only display if one or more updates requires a
restart.

> Software Updates have been installed and this Mac will restart in 5 minutes.
> Please save anything you are working on and log out by choosing Log Out from
> the bottom of the Apple menu. If you have any questions, please contact the
> helpdesk at <phone number>.

{{ figure(src="/img/wait-is-my-mac-up-to-date/p-updates-interaction.png") }}

Save your policy and we're ready to test!

When users run the "Install macOS Updates" policy, Jamf Pro will instruct the
client to download and install any available Apple Software Updates. If any of
those updates require a system restart, the user will be alerted and system will
restart as required.

## End Result

If a device has any available Software Updates, the "Install macOS Updates"
policy will be visible in Self Service. Users can run this policy to install all
available updates.

{{ figure(src="/img/wait-is-my-mac-up-to-date/self-service-available.png") }}

Once updates are installed, the device will fall _out_ of the "Software Update -
Updates Available" Smart Group and _into_ the "Software Update - No Updates
Available" group. Then, the "No macOS Updates Available" informational policy
will be visible in Self Service, and the "Install macOS Updates" will no longer
appear until Apple releases new updates.

{{ figure(src="/img/wait-is-my-mac-up-to-date/self-service-none-available.png") }}

## Resources

If you like my giant bright Self Service icons, feel free to use them.

{{ figure(src="/resources/macOSUpdates/install-macos-updates.png", max_width=256) }}

{{ figure(src="/resources/macOSUpdates/no-macos-updates-available.png", max_width=256) }}
