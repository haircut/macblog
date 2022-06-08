+++
title = "Don't disable accessibility options during Setup Assistant"
date = 2020-11-27
path = "dont-disable-accessibility"
aliases = [
    "post/dont-disable-accessibility"
]
description = "You really shouldn't skip the new 'Accessibility' pane in macOS Setup Assistant."
[extra]
author = "Matthew Warren"
+++

macOS Big Sur includes a new screen during Setup Assistant: Accessbility. It
prompts users to explore the accessibility features of macOS to adapt their
computer to their vision, motor, hearing, and cognitive needs.

You might want to disable or skip this setup screen. Don't.

<!-- more -->

{{ figure(src="/img/macos-big-sur-accessibility-setup-assistant.png") }}

The first time you boot a Mac, it runs through a process known as Setup
Assistant. This lets your users configure some basic options before they begin
using the computer. With each new macOS release, Apple adds additional screens
to Setup Assistant. Some are of dubious utility. 

Most popular MDMs allow you, as an administrator, to instruct Macs to skip
certain screens during Setup Assistant. In a corporate environment, your users
may not need to set up Apple Pay, or agree to the Terms and Conditions – so it
makes sense to simply skip these screens and streamline their first boot.

<mark><strong>If you support more humans than only yourself, how those humans
interact with their computer is not your decision.</strong></mark>

Even those who would not traditionally consider themselves in need of
accessibility features can still benefit. Putting these options front and center
allows everyone the opportunity to explore settings that might just make their
Mac a bit easier to use.

Kudos to Apple for making accessibility a _default_ aspect of the macOS setup
process.
