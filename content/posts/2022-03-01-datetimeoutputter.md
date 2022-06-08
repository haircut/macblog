+++
title = "Output the Date and Time in AutoPkg Recipes"
date = 2022-03-01
path = "autopkg-datetime"
aliases = [
    "posts/output-datetime-autopkg",
    "datetime"
]
description = "An AutoPkg processor to utput the current date and time, and optionally output future or past dates and times for use in other processors."
[extra]
author = "Matthew Warren"
+++

I recently needed to use the date and time of an AutoPkg run from within the
context of recipe.

While AutoPkg itself is aware of the date and time of a run, that information is
not accessible to other processors within the recipe.

To fill this need, I wrote a new AutoPkg processor: **DatetimeOutputter**.

**DatetimeOutputter** helps you reference the current date and time as a
variable within your AutoPkg recipes. Additionally, it can calculate future and
past dates to enhance advanced workflows.

<!-- more -->

## Current date and time

At its core, **DatetimeOutputter** adds a `datetime` variable to an AutoPkg
recipe execution. The datetime is outputted in ISO 8601 format relevant to the
local time of the computer running the recipe.

Calling the processor using [Shared Processor][sharedprocessor] syntax, with no
arguments, invokes this default behavior. Here are examples in both XML and YAML
recipe format.

XML:

```xml
<key>Process</key>
<array>
    <dict>
        <key>Processor</key>
        <string>com.github.haircut.processors/DatetimeOutputter</string>
    </dict>
</array>
```

YAML:

```yaml
Process:
  - Processor: com.github.haircut.processors/DatetimeOutputter
```

This will output a new `datetime` variable you can use in subsequent processors.
`datetime` will look something like `2022-03-01T09:45:45.544391` by default.

You can control the output format by supplying a `datetime_format` argument. The
processor accepts any [strftime][strftime]-compatible format.

Additionally, you can pass the `use_utc` argument as `True` to output the
datetime relative to UTC rather than your computer's local time.

This example illustrates both concepts:

```yaml
Process:
  - Processor: com.github.haircut.processors/DatetimeOutputter
    Arguments:
      datetime_format: "%A, %B %e, %Y at %l:%M %p"
      use_utc: True
```

This would output something like `Tuesday, March 1, 2022 at 1:45 PM` based on
UTC time.


## Past and future dates

It's handy to know when a recipe runs, but determining past and future dates –
e.g. one week from now, or 3 days ago – can augment many workflows.

**DatetimeOutputter** can generate an arbitrary number of datetime "deltas." A
datetime delta adds or subtracts time from the current datetime and outputs a
variable in your preferred format.

Outputting a delta requires specifying a few arguments.

- `output_name`: the output variable name for the time delta. Whatever you set
  here is the variable name you can use in subsequent processors.
- `direction`: specify whether the generated datetime should be in the `future`
  or `past`.
- `format`: the [strftime][strftime]-compatible format to output the datetime.
- `interval`: a dictionary of keys specifying one or more values to add or
  subtract from the current datetime. Valid keys are: `weeks`, `days`, `hours`,
  `minutes`, `seconds`, `microseconds`, and `milliseconds`.

The `interval` argument accepts any number of keyword arguments compatible with
[Python's timedelta object][timedelta], as listed above. This means you may need
to convert far-future or past intervals to "weeks" or "days;" e.g. instead of
specifying `months: 2`, use `weeks: 8`.

Here's an example of outputting two deltas:

```yaml
- Processor: com.github.haircut.processors/DatetimeOutputter
  Arguments:
    deltas:
      - output_name: one_week_in_future
        direction: future
        datetime_format: "%Y-%m-%d 00:00:00"
        interval:
          weeks: 1
      - output_name: two_days_six_hours_twelve_minutes_ago
        direction: past
        datetime_format: "%A, %B %e, at %Y %H:%M:%S"
        interval:
          days: 2
          hours: 6
          minutes: 12
```

This would output two new variables as follows:

- `one_week_in_future` with the value `2022-03-08 00:00:00`
- `two_days_six_hours_twelve_minutes_ago` with the value `Sunday, February 27,
  2022 at 03:33:45`

You can output as many deltas as you need by simply adding new dictionaries to
the `deltas` array of the processor. Each will be available for use by later
processors via variable substitution.

Deltas always respect the `use_utc` option of the processor, if set. In a single
run of **DatetimeOutputter** you cannot output UTC- and non-UTC-based datetimes.
If you have a need for this, run **DatetimeOutputter** multiple times in your
recipe, setting `use_utc` accordingly.

## Example use cases

### Keep it simple with just the date and time

In the simplest use case, you might wish to note the date a package was uploaded
by [JamfUploader][ju] within the Jamf Pro package notes field. Call
**DatetimeOutputter** first to generate the `datetime` variable, then reference
that variable as `%datetime%` in the next processor like so:

```yaml
Process:
  - Processor: com.github.haircut.processors/DatetimeOutputter
  - Processor: com.github.grahampugh.jamf-upload.processors/JamfPackageUploader
    Arguments:
      pkg_notes: "Uploaded via AutoPkg %datetime%"
```

### Output a future datetime for use with Munki's "force install after date" feature

Munki includes a feature to [force installation of a package after a specified
date][fiad]. Setting the `force_install_after_date` key in a package's `pkginfo`
dictionary will cause client Macs to forcefully logout or restart to install the
software after that (local) date and time.

If you use this functionality, Munki recommends setting the
`force_install_after_date` to at least 2 weeks in the future.

**DatetimeOutputter** can help you generate this future date in the required
format:

```yaml
- Processor: com.github.haircut.processors/DatetimeOutputter
  Arguments:
    format: "%Y-%m-%dT%H:%M:%SZ"
    deltas:
      - output_name: force_after
        direction: future
        interval:
          weeks: 2

- Processor: MunkiImporter
  Arguments:
    MUNKI_REPO: <path to Munki repo>
    pkg_path: "%pkg%"
    pkginfo:
      force_install_after_date: "%force_after%"
```

This will automate the calculation of a two-week window after uploading the
package during which users will be prompted – but not forced – to install the
package. After `force_date` passes, users will be forced to install the
software.

### Output a future date for use with a Jamf Pro policy activation date

You might wish to delay the installation of an application update for some
number of days after release. One way to do this is to configure a Jamf Pro
policy to include the "Activation Date/Time" server-side limitation. This
prevents the policy from running before the configured date and time.

**DatetimeOutputter** can generate a dynamic datetime stamp _n_ number of days
in the future. This means the new version of an application will be uploaded to
your Jamf Pro server, and the installation policy ready to go when the
Activation Date/Time rolls around. Automatically!

First, we add an `activation_date` key to the `date_time_limitations` of a
policy template. We populate the key with an `%activation_date%` substitution
variable we will later define within the associated AutoPkg recipe.

`Install-after-1-week.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<policy>
    <general>
        <name>Install %NAME% 1 Week After Release</name>
        <enabled>true</enabled>
        <trigger>CHECKIN</trigger>
        <trigger_checkin>true</trigger_checkin>
        <frequency>Once per computer</frequency>
        <date_time_limitations>
            <activation_date>%activation_date%</activation_date>
        </date_time_limitations>
    </general>
    <scope>
        <all_computers>false</all_computers>
        <computer_groups>
            <computer_group>
                <name>%UPDATE_GROUP_NAME%</name>
            </computer_group>
        </computer_groups>
    </scope>
    <self_service>
        <use_for_self_service>false</use_for_self_service>
    </self_service>
    <package_configuration>
        <packages>
            <size>1</size>
            <package>
                <name>%pkg_name%</name>
                <action>install</action>
            </package>
        </packages>
    </package_configuration>
</policy>
```

Next, we'll use **DatetimeOutputter** to generate a datestamp 1 week in
the future from the time of the AutoPkg run, in the format Jamf Pro expects for
midnight on that date. We set the `output_name` to `activation_date` to match
what the policy template expects.

```yaml
- Processor: StopProcessingIf
  Arguments:
    predicate: "pkg_uploaded == False"

- Processor: com.github.haircut.processors/DatetimeOutputter
  Arguments:
    deltas:
      - output_name: activation_date
        direction: future
        datetime_format: "%Y-%m-%d 00:00:00"
        interval:
          weeks: 1

- Processor: com.github.grahampugh.jamf-upload.processors/JamfPolicyUploader
  Arguments:
    policy_name: "Install %NAME% 1 Week After Release"
    policy_template: "Install-after-1-week.xml"
```

This will create a policy that installs the uploaded package, but remains
inactive until 1 week in the future. At that future date, the policy will
automatically activate and install on eligible Macs.

## More info

These are just a few of the potential use cases for this processor. Please check
the full details in the [README][readme]. And as always – pull requests are open
if you have an improvement or correction!

[macadmins]: <https://macadmins.org>
[sharedprocessor]: <https://github.com/autopkg/autopkg/wiki/Processor-Locations#shared-recipe-processors>
[ju]: <https://github.com/grahampugh/jamf-upload/wiki/JamfUploader-AutoPkg-Processors>
[strftime]: <https://strftime.org/>
[timedelta]: <https://docs.python.org/3/library/datetime.html#timedelta-objects>
[fiad]: <https://github.com/munki/munki/wiki/Pkginfo-Files#force-install-after-date>
[readme]: <https://github.com/autopkg/haircut-recipes/blob/master/Processors/README.md>
