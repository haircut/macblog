+++
title = "Creating a Naming Scheme for Jamf Pro"
date = 2017-12-06
path = "naming-scheme"
aliases = [
    "post/creating-a-naming-scheme-for-jamf-pro",
    "posts/creating-a-naming-scheme-for-jamf-pro/"
]
description = "Name items in your Jamf Pro instance consistenty as a favor to your colleagues and your future self."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/3"
+++

For maintaining a consistent, repeatable and intuitive administrative workflow,
you need a naming scheme for objects in your Jamf Pro Server.

Here's my take on a good strategy.

<!-- more -->

As I add more policies, Smart Groups, configuration profiles and packages to my
Jamf Pro Server, a consistent naming scheme provides clarity. If a quick glance
tells me most of what I need to know about an object, my tasks become
<mark>quicker</mark> and <mark>less error-prone.</mark>

## Focus on Function and Semantics

The name of any JPS object should quickly describe _what it does_ or _what it's
for_.

I use prefix "tags" for most object names. Policies that install apps are
prefixed `Install - `. Smart Groups containing computers that have a certain app
installed are prefixed `Has App - `.

This prefix tag visually gathers similar groups in the alphabetically-sorted JPS
web interface. All my Smart Computer Groups prefixed "Has App - " appear
together when browsing the list.

{{ figure(src="/img/namingscheme-prefixgrouping.png") }}

Using prefixes also provides the fringe benefit of making complex API operations
easier to handle. For example, I can query a particular computer and quickly
filter the `computer_group_memberships` array for items that begin with
`Outdated App - ` for a simple way to list which apps on a computer are out of
date.

### Allowed Characters in Jamf Object Names

Per Jamf support, the following special characters are allowed in object names:

`!@#$%^&*()_+-={}|[]\:";'<>?,./`

As always, be conservative, and test!

## Groups

I name most computer and user groups <mark>based on what policies or profiles I
might scope to them.</mark>

| Group Name | Scoping Intention |
| ---------- | ----------------- |
|Does Not Have App - Firefox | A policy to install Firefox|
|Outdated App - Firefox| A policy to update Firefox to the latest version|
|Has App - NoMAD | A configuration profile to deliver configuration settings for NoMAD|
|Software Update - Updates Available | A policy to install all available Apple Software Updates|
|Reject - Microsoft Office 2016 | Users who prefer LibreOffice or OpenOffice and do not want Microsoft Office on their Mac|
|Licensed - Adobe Creative Cloud | Users with an Adobe CC Subscription|

Prefixing group names with descriptive "tags" makes for a very readable and
functional policy list.

{{ figure(src="/img/namingscheme-policy-list.png") }}

The displayed information shows exactly which subset of your inventory a policy
applies to.

<mark>Avoid using Smart Groups to build reports on your inventory.</mark> This
is more appropriate for saved Advanced Searches. In most cases I avoid creating
a Smart Group unless I intend to use it in scoping. Smart Group calculation is
an expensive database operation within the JPS, so limit your total number of
Smart Groups.

## Policies

Jamf Pro 10 adds the much-welcomed ability to set both an internal policy name
and a user-facing "Display Name" shown in Self Service.

I name policies based on what they do.

| Policy Name | Display Name | Action | Scope |
| ----------- | ------------ | ------ | ----- |
| Install - Firefox | Firefox | Installs Firefox if not present on system | Does Not Have App - Firefox |
| Update - Firefox  | Update Firefox | Updates Firefox to latest version if currently installed version is outdated | Outdated App - Firefox |
| Remove - Firefox  | Remove Firefox | Removes Firefox if installed on system | Has App - Firefox |

With a sensible naming scheme, a single policy can provide clarity for your
users and sanity for you as an administrator.

{{ figure(src="/img/namingscheme-flow.png") }}

## Configuration Profiles

I name configuration profiles based on what they do or affect.

| Profile Name | Purpose |
| ------------ | ------- |
|App Defaults - Microsoft Office | Initial settings for Microsoft Office |
|Security - Login Window | A profile to disable automatic login and set the `LoginwindowText` banner |

## Packages

Most of my packages are built via [autopkg](https://github.com/autopkg/autopkg)
recipes, and follow a general `<name>-<version>.pkg` pattern. If a
vendor-provided or autopkg-created package deviates from this pattern I'll
rename it.

`Microsoft_Word-15.40.171108_Installer.pkg` is a lot more descriptive than `Word2016.pkg`.

## Conclusion

Most of the specifics are personal choices; if you prefer a prefix of `App
Installed: <name>` over `Has App - <name>`, go for it! The takeaway point is to
<mark>leverage patterns</mark> in whatever nomenclature you develop. The less
you have to hunt for a policy or scope, the more attention you can pay to the
details of accomplishing your goals with Jamf Pro.

These are my suggestions, and they work for me and my organization. Hopefully
the thinking behind why I've made certain choices is informative and helpful!
