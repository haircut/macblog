+++
title = "Renaming Computers via Snipe-IT using Jamf Pro"
date = 2021-01-13
path = "jamf-snipe-rename"
aliases = [
    "post/jamf-snipe-rename",
    "posts/jamf-snipe-rename/"
]
description = "How to integrate Jamf Pro with Snipe-IT to set the computer hostname to match your inventory management system."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/11"
+++

A couple of years ago, I shared a [method to set a Mac's hostname via a Google
Sheet][0]. It's worked well at my organization (as well as many others!) and
helped us keep our computer names consistent.

We've since moved to using [Snipe-IT][1] for asset management. Snipe is a
fantastic open-source tool that simplifies inventory tracking for our whole IT
shop. It also includes a robust API that allows us to integrate with external
systems and processes.

I'm now using the Snipe API to script our computer naming process. We treat
Snipe as the system of record for all inventory, and any change made to a
computer's hostname in Snipe can be reflected on _both_ the client system and in
Jamf Pro. Here's how.

<!-- more -->

## Why Python?

For many years, Apple included Python 2.7 with the default installation of
macOS. With the 2019 release macOS Catalina, of [Apple began warning][2] that
"future versions of macOS will not include" runtimes for popular languages
including Python.

As of macOS Big Sur 11.1, the Python 2.7 runtime is still present. However,
[Python 2 is no longer maintained][3] and is not a good choice for forward
compatibility. Many administrators are moving their system administration
processes to shell scripts, which mostly "run anywhere" and don't rely on any
runtime dependencies.

Despite the convenience and portablity of shell scripts, I maintain a number of
complex Python workflows that are arduous or ill-suited to port to shell.

This script interfaces with the Snipe API, which accepts and returns data in
JSON format. While tools and workarounds exist to work with JSON in shell
scripts, it's frankly painful.

In my organization, I've taken to installing Python 3 to a known location using
Greg Neagle's [Relocatable Python][4] tool. This allows me to use Python's
native JSON tooling and interact with the Snipe API without hassle.

As such, this script is written in Python 3, and the shebang on line `1`
requires you to specify the path to the Python 3 runtime you've installed on the
client system.

## The script

I've published this script is as a project on Github at
[haircut/jamf-snipe-rename][11].

The `set_computer_name.py` script is what you're looking for.

### What it does

The script gathers the serial number of the computer upon which it's running,
then queries your Snipe instance via the API to discover its current Asset Name
as set within that system. Then, it sets the hostname of the computer using the
Jamf binary's `setComputerName` command.

As a fallback, if your Snipe instance does not have a record of the serial
number, the computer name will be set to that serial number. Setting the
hostname to the computer's serial number is a reasonable alternative, as it lets
us easily identify machines that could not be renamed by the script, then fix
the source data within Snipe. And, the serial number is much better than a bunch
of Macs named "Sarah's MacBook Pro!"

## Setting up Snipe

I recommend setting up a dedicated local user with restricted access in Snipe to
handle all requests associated with this workflow. 

The Snipe documentation explains [creating users][5] and [setting
permissions][6]. I name my user something relevant like "Jamf Computer Naming
User" or similar.

Following the principle of least privilege, grant this local user _only_ the
**View** permission under **Assets** and **Create API Keys** under **Self**.

Log in to Snipe as the new local user, and [generate an API key][7]. This will
generate a (very long) [bearer token][8] we'll use to authenticate our script's
requests.

## Using the script

Save a local copy of `set_computer_name.py` and configure the `SNIPE_SERVER`
value (near the top of the script) to point toward your Snipe installation. It's
important to only provide the base URL here, as the script will add the full
path to the API endpoint we need.

### Encrypting script parameters

Since we're using the sensitive API key in our script to authenticate requests
to Snipe, it's best not to hardcode that API key into the script. Nor is it wise
to pass the API key as a [script parameter][9] in cleartext.

While the restricted permissions of the Snipe user limit the potential damage
from a leaked key, it's still best to avoid the hazard by encrypting the user's
API key and passing it as a script paramter.

I'm using Jamf's [Encrypted Script Parameters][10] utility to securely pass the API token from Jamf Pro to the client system. A full explanation of the utility is beyond this article's scope, but briefly:

1. Follow Jamf's README to encrypt your API key.
2. Note the ecrypted value, salt, and passphrase. Save them all in your
   organization's shared password manager :)
3. Paste the salt and passphrase in the `TOKEN_SALT` and `TOKEN_PASSPHRASE`
   constants near the top of your copy of the script.

### Adding the script to Jamf Pro

Within your Jamf Pro instance, visit _Management Settings > Computer Management
> Scripts_, then add a new script. Name it something helpful, like "Set Computer
Name from Snipe" and then set the label for Parameter 4 to "Encrypted API Key."
Paste in your (configured) copy of the script, then save.

### Running the script via policy

Create a new policy, then configure the "Script" payload. Add the "Set Computer
Name from Snipe" script, then fill in the "Encrypted API Key" script parameter
with the encrypted API key you created earlier.

I recommend updating the computer's inventory after renaming it, so that your
Jamf instance is updated with the new hostname as soon as possible. If your
deployment is large, consider omitting the inventory update to save on the
processing burden inherent in updating inventory.

How and when to run the script is largely dependent on your needs. I run the
renaming process early in our provisioning workflow using Jamf's "Enrollment
Complete" trigger.

You could also create a secondary policy that runs on a routine basis – say,
weekly – to ensure Jamf data stays consistent with your inventory data in Snipe.

### Running the script outside of Jamf Pro

If your organization does not use Jamf Pro to manage Macs, you can still use
this script after a few modifications.

As written, the script uses the Jamf binary's `setComputerName` command. This is
a convenient shorthand for running the following three commands:

```sh
sudo scutil --set ComputerName "name"
sudo scutil --set LocalHostName "name"
sudo scutil --set HostName "name"
```

Within this [project's repository][11], I've also shared
`set_computer_name_non_jamf.py` which substitues the Jamf binary call for the
three commands above. This non-Jamf script expects an _encrypted_ Snipe API
token as its first positional parameter. If you're using this workflow with a
management platform other than Jamf, I've left it as an exercise to the reader
on passing the parameter. Even if you're not using Jamf, their [encrypted script
parameters][10] project is platform agnostic.

## Conclusion

This transition from my previous [Google Sheets-based workflow][0] has been a
big help in my organization. It allows us to treat our inventory system as our
source of truth for computer names, as it should be. Changes made in the
inventory system are reflected on the client system with no manual intervention.


[0]: https://www.macblog.org/post/automatically-renaming-computers-from-a-google-sheet-with-jamf-pro/
[1]: https://snipeitapp.com/
[2]: https://developer.apple.com/documentation/macos-release-notes/macos-catalina-10_15-release-notes#:~:text=Scripting%20Language%20Runtimes
[3]: https://www.python.org/doc/sunset-python-2/
[4]: https://github.com/gregneagle/relocatable-python
[5]: https://snipe-it.readme.io/docs/managing-users
[6]: https://snipe-it.readme.io/docs/permissions
[7]: https://snipe-it.readme.io/reference#generating-api-tokens
[8]: https://oauth.net/2/bearer-tokens/
[9]: https://docs.jamf.com/10.26.0/jamf-pro/administrator-guide/Scripts.html
[10]: https://github.com/jamf/Encrypted-Script-Parameters
[11]: https://github.com/haircut/jamf-snipe-rename
