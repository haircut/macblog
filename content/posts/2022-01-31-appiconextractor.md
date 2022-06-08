+++
title = "Automatically Export and Generate App Icons in AutoPkg Recipes"
date = 2022-01-31
path = "autopkg-icons"
aliases = [
    "posts/appiconextractor",
    "icons"
]
description = "AppIconExtractor examines an app and exports its icon as a PNG image file (reading the CFBundleIconFile property from an app's Info.plist and saving that image as a PNG file. Additionally, AppIconExtractor can create icon variations by compositing a secondary image on top of the app's icon."
[extra]
author = "Matthew Warren"
+++

I'm a stickler for including icons for all policies available in Jamf Pro's Self
Service app. They help users find items in Self Service, and generally make the
app easier to use.

However, I don't like manually extracting icons from apps. It's easy enough with
a tool like [SAP's Icons app][sapicons], but if I'm automating package and
policy creation with AutoPkg, I should similarly be able to automate icon
creation, right?

I created the [**AppIconExtractor**][aie] AutoPkg processor to fully automate
this task.

At it's core, **AppIconExtractor** examines an app and exports its icon as a PNG
image file.

More technically, it reads the `CFBundleIconFile` property from an app's
`Info.plist` and saves that image as a PNG file at the path of your choice.

Additionally, `ApplIconExtractor` can create icon variations by compositing a
secondary image on top of the app's icon. This makes it simple to automatically
create a version of an icon with a destructive "red X" icon superimposed over
the app icon for use in uninstallation policies, or a version with an "update"
graphic for use in policies that update an app.

{{ figure(src="/img/appiconextractor/all-icons.png", alt="Examples of the unmodified app icon, 'install', 'update', and 'uninstall' composited icons.") }}

<!-- more -->

## Add my recipes and install the Pillow library

First, you'll need my recipe repository available to your local AutoPkg
installation. Add it with `autopkg repo-add haircut-recipes`.

**AppIconExtractor** requires installation of the [Pillow][pillow] Python
library.

Pillow is used to convert and composite icons, and can be easily installed on 
the Mac you use to run AutoPkg. Use this command:

```shell
/usr/local/autopkg/python -m pip install --upgrade Pillow
```

Note that this installs the Pillow library _within the path of AutoPkg's Python
framework_. **This is very important.** If you just run `pip` or `pip3` without
the explicit path to AutoPkg's Python installation, AutoPkg won't be able to
find the library. Recipes will produce an error directing you to install Pillow
using the specifc command above.

{% aside() %}

The <a
href="https://github.com/autopkg/autopkg/wiki/Processor-MunkiImporter">MunkiImporter
processor</a>, via munkilib, includes code to extract and convert icons from app
bundles. We could use features of this existing processor to accomplish some of
our needs. However, MunkiImporter does not provide any features to composite
images. Since <strong>AppIconExtractor</strong> is primarily targeted towards
folks managing Macs with tools other than Munki, it makes more sense to require
only a single external dependency – Pillow – that fulfills all our needs.

{% end %}

With Pillow installed, you're ready to go.

## Basic use

Using **AppIconExtractor** is as simple as including the processor as a step in a
recipe's Process dictionary. Use the [shared processor][sharedprocessor] syntax
to call `com.github.haircut.processors/AppIconExtractor`.

It requires only one argument: `source_app`, which is the path to the `.app`
from which to extract an icon. If the path to the app points inside a disk
image, that .dmg will be mounted automatically.

By default, the app's icon will be output to the recipe's cache directory as
`%NAME%.png%`. You can optionally override this output path (and filename) by
setting the `icon_output_path` argument.

A simple example in XML format might look like:

```xml
<key>Process</key>
<array>
    <dict>
        <key>Processor</key>
        <string>com.github.haircut.processors/AppIconExtractor</string>
        <key>Arguments</key>
        <dict>
            <key>source_app</key>
            <string>%RECIPE_CACHE_DIR%/CoolApp/CoolApp.app</string>
        </dict>
    </dict>
</array>
```

This will extract the icon from "CoolApp.app" and save it as a 256px square PNG
image to the recipe cache directory.

As mentioned, adding the `icon_output_path` argument will give you additional
control over the output path and filename. Here's an example in YAML format:

```yaml
Process:
  - Processor: com.github.haircut.processors/AppIconExtractor
    Arguments:
      source_app: "%RECIPE_CACHE_DIR%/CoolApp/CoolApp.app"
      icon_output_path: "%RECIPE_CACHE_DIR%/Icons/Icon-%NAME%.png"
```

## Generating composited variations

Beyond extracting the app's icon, **AppIconExtractor** can also create variation
images by compositing a "template image" on top of the app icon.

The processor can output variations for an "uninstall," "update," and "install"
version of the app icon.

{{ figure(src="/img/appiconextractor/example-composited-icons.png", alt="Examples of 'install', 'update', and 'uninstall' composited icons.") }}

To generate a variation, add a processor argument to set an output path for that
variation. Use one or more of the following arguments:

- `composite_install_path`
- `composite_update_path`
- `composite_uninstall_path`

Omit any variations you don't want to generate. The processor will only create
the variations you request be specify an output path.

If you specify only output paths for variations, **AppIconExtractor** will use
sensible defaults to composite suitable icons. The default templates are glyphs
from [SF Symbols][sfsymbols] that will work well in most situations. Each
template is 64px in size, and looks nice in the corner.

{{ figure(src="/img/appiconextractor/default-template-icons.png", alt="Default template icons for 'install', 'update', and 'uninstall' variations.") }}

These templates are encoded within the processor; <mark>you don't need to do
anything to use these defaults!</mark>

Here's an example in YAML format:

```yaml
Process:
  - Processor: com.github.haircut.processors/AppIconExtractor
    Arguments:
      source_app: "%RECIPE_CACHE_DIR%/CoolApp/CoolApp.app"
      icon_output_path: "%RECIPE_CACHE_DIR%/Icons/Icon-%NAME%.png"
      composite_update_path: "%RECIPE_CACHE_DIR%/Icons/Update-%NAME%.png"
      composite_uninstall_path: ""%RECIPE_CACHE_DIR%/Icons/Uninstall-%NAME%.png"
```

Notice that we included arguments for the "update" and "uninstall" variations,
but did not set the `composite_install_path` argument. This would output the
"bare" app icon as well as variations for "update" and "uninstall" – but no
"install" variation, since we omitted that argument.

### Custom templates

If you don't like the default variation templates, you can use your own by
setting `composite_install_template`, `composite_update_template` and/or
`composite_unsinstall_template`. Each argument should be the path to alternative
template image to use for that variation.

**AppIconExtractor** will calculate the size of the template image at the path you
specify and correctly anchor that template to the `composite_position` (see
"Padding and position" below).

{{ figure(src="/img/appiconextractor/example-custom-template.png", alt="An example of using a custom template image.") }}

Here's an example of using a custom template to generate an "uninstall"
variation in XML format:

```xml
<key>Process</key>
<array>
    <dict>
        <key>Processor</key>
        <string>com.github.haircut.processors/AppIconExtractor</string>
        <key>Arguments</key>
        <dict>
            <key>source_app</key>
            <string>%RECIPE_CACHE_DIR%/CoolApp/CoolApp.app</string>
            <key>composite_uninstall_template</key>
            <string>%RECIPE_DIR%/radical-flame.png</string>
            <key>composite_uninstall_path</key>
            <string>%RECIPE_CACHE_DIR%/delete_%NAME%.png</string>
        </dict>
    </dict>
</array>
```

## Padding and position

**AppIconExtractor** includes a few additional options to customize your
composited icon variations.

- `composite_padding`: sets the number of pixels from the edge of the image the
  superimposed template image is offset. Defaults to `10` pixels.
- `composite_position`: sets the corner to which the superimposed template image
  for composited variations is anchored. Defaults to `br` for the bottom-right
  corner. You can change this to `bl` (bottom left), `ur` (upper right), or ul
  (upper left) if you prefer.

Combinations of these options are shown below with the padding highlighted in
pink:

{{ figure(src="/img/appiconextractor/padding-and-position.png", alt="Examples of using the composite_padding and composite_position arguments.") }}

Clockwise from the upper left, this example shows:

- `composite_padding` omitted (so it defaults to `10`) and `composite_position` of `ul`.
- `composite_padding` of `20` and `composite_position` of `ur`.
- `composite_padding` of `0` and `composite_position` omitted (so it defaults to `br`).
- `composite_padding` of `5` and `composite_position` of `bl`.

Setting these options applies the same settings to *all* composited variations.
This is an intentional design choice to keep the input arguments – and thus the
required code – more manageable.

## Output variables

**AppIconExtractor** sets the path(s) to the extracted app icon, and any
composited variations, as output variables during an AutoPkg run. This means you
can extract and generate icons, then immediately use those icons in subsequent
processors like [JamfPolicyUploader][jamfpolicyuploader].

The following output variables are set if (and only if) the associated
variations are requested:

- `app_icon_path`: path to the extracted, unmodified app icon. Always set.
- `install_icon_path` path to the composited "install" variation. Only set if
  this variations is requested.
- `update_icon_path` path to the composited "update" variation. Only set if this
  variations is requested.
- `uninstall_icon_path` path to the composited "uninstall" variation. Only set
  if this variations is requested.

## Example uses in recipes

Here are two examples of using **AppIconExtractor** in a child recipe or
override.

### Extract the icon from an available .app bundle

[Greg Neagle's recipe for Sublime Text 4][sublimetext4] leaves the unarchived
.app available in the recipe cache dir at `%RECIPE_CACHE_DIR%/%NAME%/Sublime
Text.app`. We'll use this to extract the Sublime Text icon using
**AppIconExtractor's** default settings.

```yaml
Process:
  - Processor: com.github.haircut.processors/AppIconExtractor
    Arguments:
      source_app: "%RECIPE_CACHE_DIR%/%NAME%/Sublime Text.app"

  - Processor: com.github.grahampugh.jamf-upload.processors/JamfPolicyUploader
    Arguments:
      policy_template: "%POLICY_TEMPLATE%"
      policy_name: "%POLICY_NAME%"
      icon: "%app_icon_path%"
      replace_icon: True
```

This extracts Sublime Text's icon without generating any composite variations,
then feeds that extracted icon to the `JamfPolicyUploader` processor. We set
`replace_icon` to `True` to ensure any change to the icon by the vendor is
automatically reflected within our Jamf policy.

### Unpacking a .pkg to extract an icon

The recipe for the [Google Chrome Enterprise package][chromepkg] downloads a
.pkg directly from the vendor, so no repackaging is needed. And while the
AutoPkg recipe unpacks the package to performe code signature verification, it
then runs the `PathDeleter` processor to clean up that operation. This means a
child recipe does not have access to a .app from which to extract an icon.

That means we'll need to do a little more work to unpack the package again so
that we can get to the app bundle. We'll also generate a custom "uninstall"
variations and override the default composition position and padding.

Here's the Process of this more complex example in XML format:

```xml
<key>Process</key>
<array>
    <dict>
        <key>Processor</key>
        <string>FlatPkgUnpacker</string>
        <key>Arguments</key>
        <dict>
            <key>destination_path</key>
            <string>%RECIPE_CACHE_DIR%/unpack</string>
            <key>flat_pkg_path</key>
            <string>%pkg_path%</string>
        </dict>
    </dict>
    <dict>
        <key>Processor</key>
        <string>PkgPayloadUnpacker</string>
        <key>Arguments</key>
        <dict>
            <key>destination_path</key>
            <string>%RECIPE_CACHE_DIR%/unpack/pkgpayload</string>
            <key>pkg_payload_path</key>
            <string>%RECIPE_CACHE_DIR%/unpack/GoogleChrome.pkg/Payload</string>
        </dict>
    </dict>
    <dict>
        <key>Processor</key>
        <string>com.github.haircut.processors/AppIconExtractor</string>
        <key>Arguments</key>
        <dict>
            <key>composite_padding</key>
            <integer>20</integer>
            <key>composite_position</key>
            <string>ul</string>
            <key>composite_uninstall_path</key>
            <string>%RECIPE_CACHE_DIR%/Icon-Uninstall-%NAME%.png</string>
            <key>composite_uninstall_template</key>
            <string>/Users/haircut/Documents/delete.png</string>
            <key>icon_output_path</key>
            <string>%RECIPE_CACHE_DIR%/Icon-%NAME%.png</string>
            <key>source_app</key>
            <string>%RECIPE_CACHE_DIR%/unpack/pkgpayload/Google Chrome.app</string>
        </dict>
    </dict>
    <dict>
        <key>Processor</key>
        <string>PathDeleter</string>
        <key>Arguments</key>
        <dict>
            <key>path_list</key>
            <array>
                <string>%RECIPE_CACHE_DIR%/unpack</string>
            </array>
        </dict>
    </dict>
    <dict>
        <key>Processor</key>
        <string>com.github.grahampugh.jamf-upload.processors/JamfPolicyUploader</string>
        <key>Arguments</key>
        <dict>
            <key>icon</key>
            <string>%app_icon_path%</string>
            <key>policy_name</key>
            <string>Install %NAME%</string>
            <key>policy_template</key>
            <string>Self-Service-Policy.xml</string>
            <key>replace_icon</key>
            <true/>
        </dict>
    </dict>
    <dict>
        <key>Processor</key>
        <string>com.github.grahampugh.jamf-upload.processors/JamfPolicyUploader</string>
        <key>Arguments</key>
        <dict>
            <key>icon</key>
            <string>%uninstall_icon_path%</string>
            <key>policy_name</key>
            <string>Uninstall %NAME%</string>
            <key>policy_template</key>
            <string>Uninstall-Policy.xml</string>
            <key>replace_icon</key>
            <true/>
        </dict>
    </dict>
</array>
```

This unpacks the Google Chrome enterprise package, extracts the unmodified app
icon, and generates an "uninstall" composite version with a custom graphic in
the upper left corner with 20px of padding.

{{ figure(src="/img/appiconextractor/chrome-custom-example.png", alt="Examples of generating an uninstall icon using a custom template") }}

The outputs of **AppIconExtractor** are then used as inputs to
JamfPolicyUploader process runs to set the icons for two different policies.
Setting the `replace_icon` argument to `True` ensures that any changes to the
icons are reflected on the Jamf Pro policies.

---

Hopefully this processor will help you extract icons without the manual work,
and spiff up those Self Service policies.

[Pull requests accepted][pr] if you fix a bug or make an improvement!

[pillow]: <https://python-pillow.org/>
[munkilib]: <https://github.com/munki/munki/tree/main/code/client/munkilib>
[sharedprocessor]: <https://github.com/autopkg/autopkg/wiki/Processor-Locations#shared-recipe-processors>
[sfsymbols]: <https://developer.apple.com/sf-symbols/>
[jamfpolicyuploader]: <https://github.com/grahampugh/jamf-upload/wiki/JamfUploader-AutoPkg-Processors>
[sublimetext4]: <https://github.com/autopkg/gregneagle-recipes/blob/master/SublimeText/SublimeText4.pkg.recipe>
[chromepkg]: <https://github.com/autopkg/recipes/blob/master/GoogleChrome/GoogleChromePkg.pkg.recipe>
[sapicons]: <https://github.com/SAP/macos-icon-generator>
[aie]: <https://github.com/autopkg/haircut-recipes/blob/master/Processors/AppIconExtractor.py>
[pr]: <https://github.com/autopkg/haircut-recipes/compare>
