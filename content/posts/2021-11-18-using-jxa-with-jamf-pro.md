+++
title = "Using JXA in Jamf Pro Scripts and Extension Attributes"
date = 2021-11-18
path = "jamf-jxa"
aliases = [
    "posts/using-jxa-with-jamf-pro/"
]
description = "Quick advice on using JavaScript for Automation (JXA) in both Scripts and Extension Attributes within Jamf Pro."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/15"
+++

Quick follow-up to my earlier guide on [using JavaScript for Automation](@/posts/2021-11-14-how-to-parse-json-macos-command-line.md).

There must be something in the air that put the topic of JXA on the minds of
the Apple community. Armin Briegel shared a [great roundup of recent JXA
work][sosx], and the #scripting channel on the [MacAdmins Slack team][slack] is
full of folks discussing new and old discoveries.

Here are a handful of additional tips.

### Excutable JXA scripts

You can run JXA directly by including the JavaScript language flag in a script's
shebang, like this:

```javascript
#!/usr/bin/osascript -l JavaScript

function run() {
	var app = Application.currentApplication();
	app.includeStandardAdditions = true;
	return app.systemInfo().cpuType;
}
```

Save the file with a name and extension you like. I've been using `.scpt`, which
is the convention suggested by Apple's own Script Editor application. Something
like `cputype.scpt` will work just fine.

Mark the file as executable by running `chmod +x cputype.scpt`. Then, you can
run it on your system by simply calling `./cputype.scpt`

This will output the CPU architecture of the computer, for example `ARM64E` or
`Intel x86-64...`

### JXA in Jamf Pro Scripts

You can drop a JXA script directly in Jamf Pro without any modification. As
long as you include the JavaScript shebang outlined above, you're good to go.

**You do not necessarily need to wrap JXA in a shell script.** It might make
sense to "shell out" to a JXA function from a shell script for some use cases,
but it is not a requirement.

Jamf [supports non-compiled AppleScript files][jamfscripts] natively, and you
can use them in your Policies like any other script.

### JXA in Jamf Pro Extension Attributes

Same deal; specify the Script type for the Extension Attribute, and include the
JavaScript language flag shebang. Ensure your function returns the required
`<result></result>` wrapper surrounding the output. You can do that with simple
[string concatenation using the plus operator][stringconcat], like this:

```javascript
#!/usr/bin/osascript -l JavaScript

function run() {
	var app = Application.currentApplication();
	app.includeStandardAdditions = true;
	return "<result>" + app.systemInfo().cpuType + "</result>";
}
```

This will output `<result>ARM64E</result>`, for example, which will show up on
your computer's inventory record during the next device inventory.

--

That's it. Hopefully this helps you store and run JavaScript for Automation
scripts.

[sosx]: <https://scriptingosx.com/2021/11/the-unexpected-return-of-javascript-for-automation/>
[slack]: <https://www.macadmins.org>
[jamfscripts]: <https://docs.jamf.com/10.0.0/jamf-pro/administrator-guide/Managing_Scripts.html>
[stringconcat]: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Addition>
[codesign]: <https://developer.apple.com/library/archive/documentation/LanguagesUtilities/Conceptual/MacAutomationScriptingGuide/SaveaScript.html#//apple_ref/doc/uid/TP40016239-CH13-SW1>
