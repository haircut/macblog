+++
title = "How to sign scripts and other custom code"
date = 2022-10-04
description = "Instructions on code signing the scripts, LaunchAgents, LaunchDaemons and other custom code you deploy to your endpoints."
path = "sign-scripts"
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/22"
+++

Code signing is a method of certifying that a program or script was created by a specific party, and has not been altered since it was signed.

While code signing is typically discussed in the context of apps (.app bundles), the same concept applies to almost any file stored on disk.
You can easily code sign custom scripts and other files you deploy on your managed Macs.

This article provides an overview of why and how you might want to do that.

## Why sign your code as a Mac administrator?

Signing your code provides a few benefits.

First, it certifies **authenticity**.
Signed code provides macOS (and your users) a mechanism to verify that a program was created by a specific trusted entity, and deployed to the Mac without modification.

Second, it verifies code **integrity**.
If a user or malicious process modifies the code, a code signature check will fail.
As an administrator, you can leverage code signature verification to detect when custom programs have been altered or tampered with.

Third, macOS continues to surface security- and privacy-related information within the macOS interface.
Things that used to be "behind the scenes," like programs an administrator might run an organizationally managed Mac, are becoming more visible.

This increased visibility is good for end users, and means we need to take an extra step as administrators to supply identity information about the programs we deploy.

## Signed scripts are not necessarily secure scripts

It's important to note that the presence of a code signature does not mean a script is "safe."
<mark>Code signing does not evaluate a script for security vulnerabilities, malicious behavior, or code correctness.</mark>
The signature only assures that the script was provided by some entity and has not been changed since it was provided.

For app bundles, Gatekeeper assesses the code signature when a user opens that app for the first time.
Changes to the bundle's contents after the time of signing will invalidate that signature.
If the signature is invalid, Gatekeeper prevents the app from running.

Gatekeeper does no such assessment for code signed scripts you might run by automation (like via `launchd`) on a managed Mac.
It offers no protection against scripts with invalid signatures.

A malicious actor could modify external files referenced by a script, alter its interpreter, or try a number of other attacks – all without invalidating that script's signature.
macOS makes no attempt to validate a script's contents or referenced resources.

Signed scripts are certainly no _less_ secure to run or deploy than unsigned scripts, but **please do not mistake code signing as a security practice**.

## What can I sign?

You can sign just about any "flat" file.
Text files; shell scripts; Python modules; photo files; PDFs; XML property list files like LaunchDaemons and LaunchAgents.

Shell scripts are the most relevant file type for most administrators, but the steps below are the same for any file type you might sign.

## Get a signing certificate

A foundational element of all public key infrastructure is trust.
A code signing certificate needs to be issued by a certificate authority that macOS trusts in order to be considered valid.

The best place to get a trusted code signing certificate is from Apple.
Members of the [Apple Developer Program][dev] can generate one of several different certificate types suitable for code signing.

I recommend the **Developer ID Application** certificate.
This certificate type includes the Code Signing extended key usage object identifier `1.3.6.1.5.5.7.3.3` required to sign code.

1. Sign in to the Apple Developer Program account portal.
2. Click "Certificates, IDs & Profiles".
3. Click the dropdown menu near the upper left labeled "iOS, tvOS, watchOS" then select "macOS".
4. Click "All" under the "Certificates" heading, then click the plus button near the upper right.
5. When prompted "What type of certificate do you need?", select "Developer ID", click Continue, then select "**Developer ID Application**."
6. Follow Apple's on-screen instructions to generate a Certificate Signing Request (CSR) and install the resultant certificate in your login keychain.

Other certificates available from Apple may also work, but the **Developer ID Application** certificate is the best choice for this task.

If you do not participate in the Apple Developer Program, other vendors may provide code signing certificates.
You can also easily [generate a self-signed code signing certificate][selfsigned] on your Mac.
However, self-signed or third-party certificates are unlikely to be trusted by default on your client Macs.

Make your life easier and use an Apple-provided code signing certificate.

## How to sign a file

You'll need to know the common name of the certificate you wish to use for code signing.
You can list the certificates installed in your keychain that are suitable for code signing with the following command:

```shell
/usr/bin/security find-identity -p codesigning -v
```

A list of code signing certificates will be displayed on screen.

```shell
  1) {hash} "Developer ID Application: MacBlog.org (TEAMID)"
     1 valid identities found
```

The SHA-1 fingerprint and common name is displayed for each valid code signing certificate in your login keychain.
Note the common name (in quotes) of the certificate you'd like to use.
You'll need to provide that exact common name as an argument to the `codesign` binary.

```shell
/usr/bin/codesign --sign "{signing identity}" \
    --identifier "{reverse-DNS string}" \
    /path/to/file.zsh
```

In practice, this would look something like:

```shell
/usr/bin/codesign --sign "Developer ID Application: MacBlog.org (SHJS42SFS32S)" \
    --identifier "org.macblog.example" \
    ~/Code/example.zsh
```

The first option, `--sign`, tells `codesign` we wish to sign a file.
We follow that option with the full name of the signing identity we wish to use.
This is one of the valid signing identities we listed earlier using the `security` command.
Typically, this will be the common name of the Developer ID certificate you generated in the Apple Developer portal.

The second option, `--identifier`, is optional but recommended.
By convention, the identifier should be a [reverse-DNS formatted][reversedns] string that uniquely identifies your program.

If you omit the `--identifier` option, the file's name (minus extension) is used as a default identifier value.
Again, the option is not required, but it helps reduce confusion.

Finally, pass the path to the file you want to sign.
After executing the command you will be prompted to provide your keychain password (assuming your signing identity is stored in the login keychain).
This authorizes the `codesign` utility to access the private key for that certificate stored in your keychain.

The file is now signed!

## Where is the signature stored?

When developing and signing a full app bundle, the resultant `.app` will contain a `MyApp.app/Contents/_CodeSignature` folder containing information about the code signature.

Single files like scripts, however, don't have a directory structure.

Instead, the signature is stored in the file's extended attributes.
To quote Apple's [macOS Code Signing in depth article][apple-codesigning-in-depth]:

> ...code signing uses extended attributes to store signatures in non-Mach-O executables such as script files.
> If the extended attributes are lost then the program's signature will be broken.
> Many file transfer techniques do not preserve extended attributes...

You can list a file's extended attributes using `xattr`:

```shell
xattr -l /path/to/file.pdf
```

Note the `com.apple.cs.CodeDirectory` and `com.apple.cs.CodeRequirements` attributes listed in the output.
These store the components of the single file's code signature.

## Verify a code signature

The `codesign` binary can also verify the integrity of a file's code signature with the `-v` or `--verify` flag.
This tool frustratingly interprets the short `-v` flag as either "verify" or "verbose" depending on the context of other passed flags.
I stick with the longer form `--verify` to avoid confusion.

```shell
/usr/bin/codesign --verify /path/to/script.zsh
```

No output (and a `0` return code) means the signature is valid and the file is unaltered since being signed.
If you're interested in output, adding a second `v` flag will provide more detail.

```shell
$ /usr/bin/codesign --verify -v /path/to/script.zsh
script.zsh: valid on disk
script.zsh: satisfies its Designated Requirement
```

Otherwise, specifics of the problem with a signature will be displayed, and the command returns an error code.

Because `codesign`'s `verify` function returns proper success or failure status codes, it's easy to validate a code signature within the context of another shell script.
Here is a small example:

```shell
if /usr/bin/codesign -v "/path/to/file.zsh"; then
    echo "Code signature is valid."
else
    echo "Code signature is invalid!"
fi
```

Your configuration management system can run checks like the above to validate the integrity of custom scripts deployed on your endpoints.
If the signature check fails, you know you need to redeploy the script.

## Delivery

Source control systems like Git will drop extended attributes from files when committed.
If you sign a script, then commit it to a Git repository, it will no longer be signed.

Additionally, your configuration management tool may or may not be able to distribute files with extended attributes – and thus code signing – intact.

Because of these limitations, I recommend you package any signed code as a final step before distribution.
Signed scripts **will** retain extended attributes and remain signed when delivered to managed Macs if you package them. 

## Common signing problems

I mentioned you can code sign pretty much any file type.
This includes images, PDFs, text files – almost anything (though in some cases, I'm not sure why you would want to).

If you try to sign an incompatible file – like a directory – you'll get an error message stating `bundle format unrecognized, invalid, or unsuitable`.

If you receive the error `resource fork, Finder information, or similar detritus not allowed`, this indicates the file has extended attributes that are incompatible with computing a code signature hash.
You may be able to discard these attributes if they are not essential to the file type. For example, the most common incompatible extended attribute is the legacy "Finder Info" stored in the `com.apple.FinderInfo` attribute. This attribute contains metadata like Finder tags and other information that is unlikely relavent for files mass deployed by an administrator.

Again, list the file's extended attributes using `xattr`:

```shell
xattr -l /path/to/file.pdf
```

Note the `com.apple.FinderInfo` attribute in the ouput.

Removing the "Finder Info" attribute is as simple as `xattr -d com.apple.FinderInfo /path/to/file.pdf`.
Or, remove *all* extended attributes via `xattr -c /path/to/file.pdf`.
You should then be able to code sign the file.

But remember – do not strip all extended attributes _after_ signing the file.
That would remove the signature!


## Further reading

I've covered the basics here, but will close with a few recommended resources for more information on code signing within macOS.

- [Apple - TN3127: Inside Code Signing: Requirements][tn3127]
- [Apple - About Code Signing][apple-about-codesigning]
- [Apple - TN2206 macOS Code Signing In Depth][apple-codesigning-in-depth]

[apple-about-codesigning]: <https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/Introduction/Introduction.html>
[apple-codesigning-in-depth]: <https://developer.apple.com/library/archive/technotes/tn2206/_index.html>
[dev]: <https://developer.apple.com/programs/>
[selfsigned]: <https://eclecticlight.co/2019/01/16/code-signing-for-the-concerned-2-creating-a-personal-certificate/>
[reversedns]: <https://en.wikipedia.org/wiki/Reverse_domain_name_notation>
[tn3127]: <https://developer.apple.com/documentation/technotes/tn3127-inside-code-signing-requirements>

