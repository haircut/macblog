+++
title = "Stopping an AutoPkg Recipe if VirusTotal Detects Malware"
date = 2021-12-14
path = "stop-autopkg-virustotal"
aliases = [
    "posts/stop-autopkg-virustotal"
]
description = "How to stop an AutoPkg run – and avoid uploading a potentially compromised package to your infrastructure – if VirusTotal detects malware."
[extra]
author = "Matthew Warren"
+++

Hannes Juutilainen's [VirusTotalAnalyzer][vta] is a fantastic AutoPkg
postprocessor. It automatically queries [VirusTotal][vt] to analyze items
downloaded by AutoPkg and detect potential malware.

VirusTotalAnalyzer was designed to run as a [postprocessor][postprocessor].
AutoPkg postprocessors allow you to add extra "steps" to an AutoPkg recipe at
runtime without modifying the recipe itself. By this convention,
VirusTotalAnalyzer scans files after all other recipe steps have completed. This
means a recipe cannot conditionally act on the VirusTotal scan results; the
query happens _after_ the recipe has otherwise finished.

In practice, [code signature verification][codesig], [recipe trust
verification][trust], and after-the-fact VirusTotal scanning offer strong
protections against malicious software. Most Mac admins also report "never"
seeing VirusTotal flag a vendor package; or if they have seen it, investigation
revealed a false positive.

However, many AutoPkg workflows directly upload or import software packages to a
Munki repository or Jamf Pro distribution point as part of a recipe run. If
VirusTotal engines flag an item, VirusTotalAnalyzer reports on the detection
*after the item is already uploaded to your systems*. Further, most
highly-automated AutoPkg workflows begin deploying the newly-uploaded software
to a test group (or all endpoints) as soon as the recipe completes.

You may require additional assurance that downloaded software is not flagged by
VirusTotal, and want to prevent any flagged files from being uploaded to your
software distribution points.

You can do this by using a custom recipe that runs VirusTotalAnalyzer as a
regular processor – instead of as a postprocessor – combined with the
StopProcessingIf processor. This allows you to terminate a recipe if VirusTotal
reports any hits **before** subsequent recipe steps upload an item to your
systems.

Here's how to do that.

<!-- more -->

### Create a Custom Recipe

While it's possible to _append_ additional processors to a recipe within a
[recipe override][override], or call multiple postprocessors on the command
line, I recommend creating a custom recipe for each application. This allows you
to very specifically define _exactly_ what you wish to happen during a recipe
run.

Yes, this is additional overhead. However, it's not uncommon for an organization
to maintain custom recipes for all software to meet automation needs. And hey,
if you want this extra security layer, you'll have to work a little for it.

I recommend creating an organization-specific child recipe for each app in your
catalog, and adding the required processors to that child recipe.

Let's use [Rectangle][rect] as an example. Here's the shell of a new custom
recipe:

```yaml
Identifier: org.macblog.pkg.Rectangle
ParentRecipe: com.github.dataJAR-recipes.pkg.Rectangle
MinimumVersion: "2.3"
Input:
  NAME: Rectangle
Process:
```

Here, I've set a new `Identifier` for the recipe specific to my organization. In
this example, I ultimately want a .pkg, so I set the `ParentRecipe` as the
identifier of the `pkg` version of the Rectangle recipe. Finally, since I'm
using YAML here, I set the `MinimumVersion` to `2.3` to require the version of
AutoPkg that supports YAML-formatted recipes.

I'm setting the `NAME` key in the `Input` dictionary to ensure AutoPkg
recognizes the file as a valid recipe. No additional Input keys are required for
this minimal example. You can, however, customize them to fit your needs.

Since this is a child recipe it inherits all the output of the parent recipe.
That means we don't need to do anything to get the pkg output from the
parent, and the child recipe only needs to contain the additional processors we
want to run.

{{ figure(src="/img/autopkg-virustotal-stopprocessingif.png", alt="Flow chart illustrating parent and child AutoPkg recipes.", caption="Each child recipe receives and can act on the output of its parent recipe.") }}

### Add VirusTotalAnalyzer as a Regular Processor

Instead of running VirusTotalAnalyzer as a postprocessor, you can call it as a
"regular" processor inside the `Process` dictionary of our custom recipe. Add it
like so:

<pre class="language-yaml z-code">
    <code>
Identifier: org.macblog.pkg.Rectangle
ParentRecipe: com.github.dataJAR-recipes.pkg.Rectangle
MinimumVersion: "2.3"
Input:
  NAME: Rectangle
<mark>Process:
  - Processor: io.github.hjuutilainen.VirusTotalAnalyzer/VirusTotalAnalyzer</mark>
    </code>
</pre>

Since processors within a recipe run sequentially, you can run
VirusTotalAnalyzer immediately after downloading an item, but before subsequent
processors upload that item to your systems. It requires only that a `pathname`
variable exists.

In this example, the parent recipes have already handled downloading and
packaging the app. Our child recipe receives all the output of those previous
actions, and we're ready to use VirusTotalAnalyzer to scan the downloaded item
for threats.

### Add a StopProcessingIf Processor

Next, add the [StopProcessingIf][spi] processor. The StopProcessingIf processor
terminates a recipe execution if a condition is met. This is most commonly
used to stop processing a recipe if a vendor download has not changed, by
setting the `predicate` to `download_changed == FALSE`.

Because StopProcessingIf accepts [NSPredicate-style][nsp] conditions, we can
also test more complex conditions.

VirusTotalAnalyzer outputs its findings to the
`virus_total_analyzer_summary_result` output variable. Within that output is a
`ratio` key that denotes the number of VirusTotal engines that flagged a file as
suspicious out of the total number of engines that scanned the file, e.g.
`0/55`.

We can use the NSPredicate test `BEGINSWITH` to target the beginning of the
`ratio` variable string. That variable is nested a couple levels deep, and we
use dot notation to access nested values. Combined with a `NOT` negation, our
full predicate is:

```
NOT virus_total_analyzer_summary_result.data.ratio BEGINSWITH '0/'
```

This means: _If the reported ratio from VirusTotal begins with anything other
than `0/`, e.g. `1/54` or `23/68`, stop processing the recipe._

Add these items to the `Input` and `Process` dictionaries like so:

<pre class="language-yaml z-code">
    <code>
Identifier: org.macblog.pkg.Rectangle
ParentRecipe: com.github.dataJAR-recipes.pkg.Rectangle
MinimumVersion: "2.3"
Input:
  NAME: Rectangle
<mark>  STOP_PREDICATE: "NOT virus_total_analyzer_summary_result.data.ratio BEGINSWITH '0'"</mark>  
Process:
  - Processor: io.github.hjuutilainen.VirusTotalAnalyzer/VirusTotalAnalyzer

<mark>  - Processor: StopProcessingIf
    Arguments:
      predicate: "%STOP_PREDICATE%"</mark>
    </code>
</pre>

Setting the predicate as an Input key allows you to override the predicate at
recipe runtime if you need to temporarily skip the StopProcessingIf
functionality. Manually passing `--key STOP_PREDICATE=FALSEPREDICATE` will bypass the
StopProcessingIf processor and allow the recipe to complete, even if VirusTotal
reports a detection.

<p class="update">
    <span class="update-lozenge">Update:</span> Thanks to <a href="https://grahamrpugh.com/" title="Graham Pugh's website">Graham Pugh</a> for the suggestion to specify the predicate in the Input dictionary.
</p>

### Add Your Other Processors

Add any additional processors _after_ the StopProcessingIf processor to suit
your needs.

You might add [JamfUploader][jamfupload] or [MunkiImporter][munkiimporter]
processors to copy a package to your distribution servers. Whatever you add
after the StopProcessingIf processor will be executed only if VirusTotal reports
no malware detections.

This custom recipe can be used just like any other. You can (and should!) make
local overrides, run it in recipe lists, add chat-notifying postprocessors, and
commit it to your organization's private recipe repository. Run it manually, or
within AutoPkgr, or in your CI pipeline.

### Final Thoughts

There are a few caveats worth mentioning:

1. You need to create a custom recipe for every app in your catalog. You may be
   doing this already – if so, good for you! (Or, try this
   [alternative](#alternative-to-a-custom-recipe))
2. You might get false positives, where a single VirusTotal engine erroneously
   reports a detection. You'll need to follow up manually, and perhaps
   temporarily disable the VirusTotalAnalyzer (or StopProcessingIf) processor in
   a recipe until VirusTotal or the vendor fixes the issue. As mentioned,
   passing `--key STOP_PREDICATE=FALSEPREDICATE` will disable the StopProcessingIf
   behavior.
3. You cannot set a "threshold" of a permissible number of potential detections.
   The predicate used in the StopProcessingIf processor allows only **zero**
   detections.

If you don't mind this overhead, and your organization requires this increased
assurance, you fit within the narrow band of utility this process provides. You
might even view the above caveats as good things. If so, enjoy!

### Alternative to a Custom Recipe

AutoPkg is flexible, and you can run multiple postprocessors sequentially.
Instead of creating a custom recipe, it's possible to "append" the required
postprocessors to an existing recipe on the command line as follows:

```bash
autopkg run com.github.dataJAR-recipes.pkg.Rectangle \
--post io.github.hjuutilainen.VirusTotalAnalyzer/VirusTotalAnalyzer \
--post StopProcessingIf \
--key predicate="NOT virus_total_analyzer_summary_result.data.ratio BEGINSWITH '0'" \
--post com.github.grahampugh.jamf-upload.processors/JamfPackageUploader
```

I personally prefer maintaining a custom recipe. I most frequently run multiple
recipes using the `-l` or `--recipe-list` flag, and adding postprocessors to a
recipe list run will call all those postprocessors for every recipe. For me,
that decreases the flexibility afforded by creating a very explicit custom
recipe, which is why I recommend that route.

But the multiple-postprocessor option works fine if you prefer it.

### Complete Examples

Here is a complete recipe example in both YAML and XML formats. It demonstrates
uploading the example Rectangle package to Jamf Pro if VirusTotal engines find
no malware.

#### YAML

```yaml
Identifier: org.macblog.pkg.Rectangle
ParentRecipe: com.github.dataJAR-recipes.pkg.Rectangle
MinimumVersion: "2.3"
Input:
  NAME: Rectangle
  STOP_PREDICATE: "NOT virus_total_analyzer_summary_result.data.ratio BEGINSWITH '0'"
Process:
  - Processor: io.github.hjuutilainen.VirusTotalAnalyzer/VirusTotalAnalyzer

  - Processor: StopProcessingIf
    Arguments:
      predicate: "%STOP_PREDICATE%"

  - Processor: com.github.grahampugh.jamf-upload.processors/JamfPackageUploader
```

#### XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Identifier</key>
  <string>org.macblog.pkg.Rectangle</string>
  <key>Input</key>
  <dict>
    <key>NAME</key>
    <string>Rectangle</string>
    <key>STOP_PREDICATE</key>
    <string>NOT virus_total_analyzer_summary_result.data.ratio BEGINSWITH '0'</string>
  </dict>
  <key>MinimumVersion</key>
  <string>2.3</string>
  <key>ParentRecipe</key>
  <string>com.github.dataJAR-recipes.pkg.Rectangle</string>
  <key>Process</key>
  <array>
    <dict>
      <key>Processor</key>
      <string>io.github.hjuutilainen.VirusTotalAnalyzer/VirusTotalAnalyzer</string>
    </dict>
    <dict>
      <key>Arguments</key>
      <dict>
        <key>predicate</key>
        <string>%STOP_PREDICATE%</string>
      </dict>
      <key>Processor</key>
      <string>StopProcessingIf</string>
    </dict>
    <dict>
      <key>Processor</key>
      <string>com.github.grahampugh.jamf-upload.processors/JamfPackageUploader</string>
    </dict>
  </array>
</dict>
</plist>
```

[vt]: <https://www.virustotal.com>
[codesig]: <https://github.com/autopkg/autopkg/wiki/Processor-CodeSignatureVerifier>
[trust]: <https://github.com/autopkg/autopkg/wiki/AutoPkg-and-recipe-parent-trust-info>
[vta]: <https://github.com/hjuutilainen/autopkg-virustotalanalyzer>
[postprocessor]: <https://github.com/autopkg/autopkg/wiki/PreAndPostProcessorSupport>
[spi]: <https://github.com/autopkg/autopkg/wiki/Processor-StopProcessingIf>
[nsp]: <http://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/Predicates/Articles/pSyntax.html>
[override]: <https://github.com/autopkg/autopkg/wiki/Recipe-Overrides>
[rect]: <https://rectangleapp.com>
[munkiimporter]: <https://github.com/autopkg/autopkg/wiki/Processor-MunkiImporter>
[jamfupload]: <https://github.com/grahampugh/jamf-upload/wiki/JamfUploader-AutoPkg-Processors>
