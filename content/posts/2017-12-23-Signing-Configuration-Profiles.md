+++
title = "Signing Configuration Profiles"
date = 2018-08-03
path = "sign-configuration-profiles"
aliases = [
    "post/signing-configuration-profiles",
    "posts/signing-configuration-profiles/"
]
description = "The definitive guide – I think? – to signing your Configuration Profiles."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/4"
+++

Apple has made it clear; MDM is the future.

As the preferred method of device management moves more and more to
Configuration Profiles, administrators must turn their focus toward digital
security.

Signing configuration profiles provides assurance of their origin, and an
assertion their contents have not been modified in transit.

<!-- more -->

A profile signed with a trusted signing certificate appears in System
Preferences > Profiles with a green "Verified" label.

{{ figure(src="/img/signing-profiles-verified.png") }}

If the profile is signed by a certificate that was not issued by a certificate
authority trusted by the operating system – as is the case with a self-signed
certificate – the profile appears with a red "Unverified" label.

{{ figure(src="/img/signing-profiles-unverified.png") }}

If the profile is not signed at all, it appears with a red "Unsigned" label.

{{ figure(src="/img/signing-profiles-unsigned.png") }}

{% aside() %}
    <p><strong>Note:</strong> While this guide focuses on signing Configuration Profiles for use with Jamf Pro, the concepts apply generally.</p>
{% end %}

Jamf Pro includes a built-in certificate authority. It's a key piece of
functionality that allows Jamf Pro to generate digital certificates to
facilitate secure, trusted communication between your clients and your Jamf Pro
server.

During setup, the Jamf Pro built-in certificate authority generates a signing
certificate for both macOS and iOS. This signing certificate is used to
digitally sign configuration profiles deployed by your Jamf Pro server. Each
enrolled client will have the "JSS Built-in Signing Certificate" installed to
its System keychain. This certificate is issued by your Jamf Pro server's
built-in certificate authority. Since your clients trust your Jamf Pro server's
root certificate, the signing certificate is also trusted; the chain is
complete.

Profiles created using the Jamf Pro configuration profile interface are
automatically signed using the "JSS Built-in Signing Certificate" created during
installation of your Jamf Pro server.

{{ figure(src="/img/signing-profiles-signed-inset.png") }}

## Why Sign Your Profiles?

If you only use Jamf Pro's configuration profile creation interface, Jamf Pro
handles all the signing for you.

However, if you create custom profiles "from scratch," then upload them to Jamf
Pro for deployment, you should sign them.

When you upload an unsigned configuration profile, the Jamf Pro server converts
it into a format you can edit using their interface. Unfortunately, this
conversion process often results in a finished profile that incorrectly manages
the settings you specified, or even manages settings that were not included in
the original source profile at all.

Here's one example. Uploading a profile that manages the domain
`com.apple.security.firewall` and sets the key `EnableFirewall` to `true`
results in a Jamf-converted profile that sets that key to `false` – the exact
opposite of what you intended! This particular defect is filed as Product Issue
PI-004130, but no resolution is currently in place.

<mark>Signing your custom configuration profiles before uploading them prevents
the Jamf Pro server from manipulating their contents.</mark> This ensures your
profile is delivered unaltered to your clients.

When you upload a signed configuration profile, Jamf Pro warns you that the
profile is read-only and offers you the opportunity to remove the digital
signature.

{{ figure(src="/img/signing-profiles-read-only.png") }}

Don't click the "Remove Signature" button. You won't be able to edit the profile
within the Jamf Pro web interface – but neither can Jamf! The profile is
delivered to client systems unmodified.

If you do need to make edits to the profile, you'll need to do so in a text
editor, re-sign the profile, then upload it again to your Jamf Pro server. It's
a minor inconvenience, but ensures you don't accidentally manage unintended
preferences.

## Signing Profiles for Trust by Any Client

With an Apple Developer certificate you can easily sign profiles (and other
objects) to provide assurance of origin. So long as you abide by the terms of
the Apple Developer Program, your profiles will be trusted on any macOS or iOS
device.

If you manage more than a handful of devices, you should **really** sign up for
the [Apple Developer Program](https://developer.apple.com/programs/). The
Developer Program will provide you with a number of certificates issued by Apple
that are trusted on all macOS and iOS devices. This is useful for signing your
Jamf Pro QuickAdd package, signing other installer packages, and signing
configuration profiles. When these objects are signed you avoid the hurdles of
your packages being quarantined by Gatekeeper, or your profiles appearing as
"unverified."

1. Sign in to the [Apple Developer Program account
   portal](https://developer.apple.com/account). 
2. Click "Certificates, IDs & Profiles".
3. Click the dropdown menu near the upper left labeled "iOS, tvOS, watchOS" then
   select "macOS".
4. Click "All" under the "Certificates" heading, then click the plus button near
   the upper right.
5. When prompted "What type of certificate do you need?", select "Developer ID",
   click Continue, then select "Developer ID Installer." While this type of
   certificate may be the best choice, other certificates will also work. See
   the note below for details.
6. Follow Apple's on-screen instructions to generate a Certificate Signing
Request (CSR) and install the resultant certificate in your login keychain.

With your certificate installed in your keychain, use the `security` tool to
sign a profile.

```shell
/usr/bin/security cms -S -N "<common name of certificate>" -i <input path to unsigned profile> -o <output path for signed profile>
```

A complete example might look like:

```shell
/usr/bin/security cms -S -N "Mac Developer: Matthew Warren (XXXXXXXXXX)" -i ~/Documents/Unsigned.mobileconfig -o ~/Documents/Signed.mobileconfig
```

When signing a Profile, you'll be asked to unlock your login keychain. Enter
your login keychain password and click "Allow".

{{ figure(src="/img/signing-profiles-keychain-unlock.png") }}

You could also click "Always Allow" to avoid future prompts when signing
Profiles, but I tend to err on the side of security and just enter my password
each time.

#### _Note: What type of certificate should you choose?_

Apple provides a number of different certificate types. While I recommend the
"Developer ID Installer," most (if not all) of the options Apple provides will
produce a certificate with the correct Key Usage attributes required to sign a
Configuration Profile.

The choice is largely aesthetic, since the Common Name of the certificate will
be visible to end users when viewing a Configuration Profile in the _Profiles_
pane within System Preferences.

The visible name listed beside the "Signed" heading generally follows the
pattern `<Certificate Type>: <Company or Individual Name> (<Team ID>)`. For
example:

{{ figure(src="/img/signing-profiles-devid.png") }}

Another consideration is the Apple ID used to create the certificate. If you're
signed in using the "parent" Apple ID of your organization's Apple Developer
Account you'll have the most options available, including the recommended
"Developer ID Installer" option. Generated certificates will include your
**company or organization's** name in their Common Name. If you're using an
Apple ID that has been granted delegated access to your organization's Developer
Program, generated certificates will instead have **your name** included in the
Common Name.

Again, most if not all of the Apple's options are suitable for signing
Configuration Profiles – just keep in mind the differences in presentation.

Next, let's take a look at an alternative way to generate a signing certificate
we can use to sign Configuration Profiles _without_ having access to Apple's
certificate portal.

## Signing Profiles for Trust Only by Jamf-enrolled Clients

If you do not (and/or _cannot_) participate in the Apple Developer Program, you
can use Jamf Pro to generate a certificate to sign your custom configuration
profiles. Profiles signed with a Jamf Pro-generated certificate will be trusted
**only** by clients enrolled to your Jamf Pro instance; however, this is likely
sufficient for most use cases.

Jamf Pro's Built-in CA allows you to generate arbitrary certificates of the
following types:

- Client Certificate
- Web Server Certificate
- SSO Certificate

During setup, the Jamf Pro built-in certificate authority generates a signing
certificate for both macOS and iOS. This signing certificate is used to
digitally sign configuration profiles deployed by your Jamf Pro server. Each
enrolled client will have the "JSS Built-in Signing Certificate" installed to
its System keychain. This certificate is issued by your Jamf Pro server's
built-in certificate authority. Since your clients trust your Jamf Pro server's
root certificate, the signing certificate is also trusted; the chain is
complete.

While it might seem tempting to re-use these certificates to sign your
custom-crafted profiles, Jamf Pro does not present a GUI option to download
them. This is a good thing!

If you _could_ download and use the private key of your "JSS Built-in Signing
Certificate" to sign arbitrary profiles it would present a larger security risk.
If that key was ever compromised, it would affect every item signed by that
certificate. This includes devices' MDM enrollment profiles. You'd have to
regenerate your Jamf Pro built-in certificate authority and re-enroll every
client. Let's avoid that!

Instead, we'll generate our own certificate for signing configuration profiles.
First we'll need to generate a Certificate Signing Request (CSR) on your Mac.

### Create A CSR on Your Mac

Perform these steps on a client enrolled to your Jamf Pro server so that the
certificate trust chain is complete.

1. Open the Keychain Access app.
2. Click the **Keychain Access** menu in the menu bar, hover over _Certificate
   Assistant_, then select _Request a Certificate From a Certificate
   Authority..._.
3. In the **Certificate Assistant** window that appears enter an email address
    for your organization (I use a generic one) and a suitable Common Name, such
    as "<name of your organization> Profile Signing Certificate" or similar. For
    the "Request is:" option, select "Saved to disk" then click _Continue_. {{
    figure(src="/img/signing-profiles-certificate-assistant1.png") }}
4. When prompted, choose a location to save the CSR.

### Upload the CSR to your Jamf Pro Server

1. Open the CSR you saved to disk during the previous steps in a plain text
editor and copy the entire contents to your clipboard. {{
    figure(src="/img/signing-profiles-csr-text.png") }}
2. Sign in to your Jamf Pro server and navigate to _Settings > Global Management
   > PKI Certificates_.
3. Click the "Management Certificate Template" tab near the top of the screen.
4. Click the "Create Certificate from CSR" button.
5. In the modal window, paste the contents of your CSR, then select a
_Certificate Type_ of "Web Server Certificate" and click the "Create" button. {{
    figure(src="/img/signing-profiles-paste-csr.png") }}
6. After clicking "Create", a file with a name in the format of `C=US,CN=<common
name you chose>,E=<email you specified>.pem` will automatically download. In my
experience, the Jamf Pro GUI freezes at this point; simply click any link on the
page to navigate away – we've already got what we came for!
7. Open the downloaded `.pem` file. When prompted, choose to install the
    certificate in your login keychain. {{
    figure(src="/img/signing-profiles-where-install.png") }}

The certificate is now installed to your login keychain and ready to reference
by name when signing Configuration Profiles.

```shell
/usr/bin/security cms -S -N "<common name of certificate you just installed>" -i <input path to unsigned profile> -o <output path for signed profile>
```

When prompted, enter unlock your login keychain to finish signing the profile.

Finally, let's look at a third way to generate a certificate suitable for
signing Profiles.

## Signing a Profile – The Quick-and-Dirty, But Not Best Way

If for some reason you neither participate in the Apple Developer Program, nor
create a certificate within your Jamf Pro server, you do still have an option.
We can create a Self Signed Certificate suitable for signing profiles. The
resulting signed profile will not be trusted once deployed to a client system,
but its signature _will_ prevent Jamf Pro from modifying its contents. Once
installed, if the profile is viewed in the Profiles pane of System Preferences,
it will display the red "Unverified" notice.

### Create a Self Signed Code Signing Certificate

1. Open the Keychain Access app.
2. Click the **Keychain Access** menu in the menu bar, hover over _Certificate
   Assistant_, then select _Create a Certificate..._.
3. In the **Certificate Assistant** window that appears, enter a sensible name.
Select the _Identity Type_ of "Self Signed Root" and _Certificate Type_ of "Code
    Signing". Check the box beside "Let me override defaults". {{
    figure(src="/img/signing-profiles-create-cert-1.png") }}
4. Leave the _Serial Number_ set to "1", and increase the _Validity Period_ to
something like "1096" days (3 years) for convenience. {{
    figure(src="/img/signing-profiles-create-cert-2.png") }}
5. On the next screen, enter relevant values for your organization. Make sure to
specify a descriptive _Common Name_ for the certificate, as this will be
    viewable when inspecting Configuration Profiles signed by this certificate.
    {{ figure(src="/img/signing-profiles-create-cert-3.png") }}
6. Leave the default values of "2048 bits" for _Key Size_ and "RSA" for
    _Algorithm_. {{ figure(src="/img/signing-profiles-create-cert-4.png") }}
7. Ensure "Signature" is selected for the _Key Usage Extension_. {{
    figure(src="/img/signing-profiles-create-cert-5.png") }}
8. Ensure "Code Signing" is selected for the _Extended Key Usage Extension_. {{
    figure(src="/img/signing-profiles-create-cert-6.png") }}
9. On the following screens, leave "Include Basic Contraints Extension" and
   "Include Subject Alternate Name Extension" unchecked.
10. When prompted, ensure the certificate is set to be stored in your "login"
    keychain. {{ figure(src="/img/signing-profiles-create-cert-9.png") }}

After clicking "Done" you should see the new certificate listed in your login
keychain.

{{ figure(src="/img/signing-profiles-in-keychain.png") }}

You can now sign Configuration Profiles using the `security` binary by
referencing the common name of your new self-signed certificate.

```shell
/usr/bin/security cms -S -N "<common name of self-signed certificate>" -i <input path to unsigned profile> -o <output path for signed profile>
```

Again, keep in mind that Configuration Profiles signed with a self-signed
certificate will be listed as "Unverified" in System Preferences. While this
won't affect the ability to install the profile, it could be a security concern
depending on your environment. "Unverified" profiles assure that their contents
have not been modified in transit, but provided no assurance of origin.

## GUI Tool

If you'd prefer a graphical tool to manage signing Configuration Profiles (or
packages), check out [Hancock](https://github.com/JeremyAgost/Hancock). You'll
still need to correctly generate a signing certificate using a method outlined
above, but Hancock can save you some keystrokes if you prefer an app!

## Wrapping Up

We've covered three different methods – in order of preference – to generate a
signing certificate we can use to sign Configuration Profiles.

Signing a Configuration Profile prevents an MDM system, like Jamf Pro, from
tampering with its contents; your hand-crafted profile is delivered to clients
unaltered. If your signing certificate is trusted by the client system, it also
provides verification that the Configuration Profile was created and delivered
by a trusted party.

Regardless of how you generate your signing certificate, remember the command:

```shell
/usr/bin/security cms -S -N "<common name of signing certificate>" -i <input path to unsigned profile> -o <output path for signed profile>
```

That's it! Don't hesitate to reach out if you have any questions, feedback, or
corrections.
