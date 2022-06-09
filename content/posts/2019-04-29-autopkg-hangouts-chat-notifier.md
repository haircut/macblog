+++
title = "Sending Autopkg and JSSImporter Notifications to Google Hangouts Chat"
date = 2019-04-29
path = "autopkg-google-chat"
aliases = [
    "post/autopkg-hangouts-chat-notifications",
    "posts/autopkg-hangouts-chat-notifier"
]
description = "Two AutoPkg processors to send you the results of an AutoPkg run in Google Chat."
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/9"
+++

Although [Slack](https://slack.com) has seemingly taken over the world
of workplace chat, my organization is a **G Suite** shop and we use 
[Hangouts Chat](https://gsuite.google.com/products/chat/) for a majority
of our internal communication. It's included as a "core" G Suite app,
so why not use the product we already have, right?

I wanted a way to post notifications to Hangsouts Chat rooms when
[autopkg](https://github.com/autopkg/autopkg) downloads new software, or makes
changes to our Jamf Pro server via
[JSSImporter](https://github.com/jssimporter/JSSImporter). No solution existed.
Building on the excellent Slack-centric work of both [Graham R
Pugh](https://grahamrpugh.com/2017/12/22/slack-for-autopkg-jssimporter.html) and
[Rich
Trouton](https://derflounder.wordpress.com/2018/07/06/automating-autopkg-runs-with-autopkg-conductor/),
I've made two different [autopkg
postprocessors](https://github.com/autopkg/autopkg/wiki/PreAndPostProcessorSupport)
to send autopkg notifications to Google Hangouts Chat.

<!-- more -->

### `HangoutsChatNotifier` - simple notifications

If you'd just like to know when your automated autopkg run downloads new
software, `HangoutsChatNotifier` will send a simple note to a Hangouts
Chat room.

{{ figure(src="/img/autopkg-hangouts-chat-notifications/simple-notification.png") }}

### `HangoutsChatJSSNotifier` - going deeper with `JSSImporter`

If you're using [JSSImporter](https://github.com/jssimporter/JSSImporter) to
automate the process of adding the software autopkg downloads to your JSS,
`HangoutsChatJSSNotifier` will send more detailed messages to a Hangouts Chat
room.

{{ figure(src="/img/autopkg-hangouts-chat-notifications/jss-notification.png") }}

Both postprocessors leverage the existing data provided during an autopkg run to
send data to a Hangouts Chat [incoming
webhook](https://developers.google.com/hangouts/chat/how-tos/webhooks).

To set these up on your own G Suite domain, you'll need:

- [Hangouts Chat](https://gsuite.google.com/products/chat/) enabled for your
  users
- [Bots enabled](https://support.google.com/a/answer/7651360) for all OUs in
  your domain. If Bots are not enabled across the domain, your notifications
  [will
  fail](https://developers.google.com/hangouts/chat/how-tos/webhooks#limitations)
- My [haircut-recipes](https://github.com/autopkg/haircut-recipes) autopkg
  recipes present in your autopkg repository, which includes these Hangouts Chat
  postprocessors
- An [incoming
  webhook](https://developers.google.com/hangouts/chat/how-tos/webhooks)
  configured for the Hangouts Chat room you wish to notify
- ...and of course, a working autopkg setup

## Setting up a webhook

In Hangouts Chat, open the room you'd like to notify, then click the room's name
near to top of the screen.

{{ figure(src="/img/autopkg-hangouts-chat-notifications/configure-webhooks.png") }}

Click _Configure webhooks_. Enter a name for the incoming webhook – I just
use "autopkg" – then click _Save_. 

{{ figure(src="/img/autopkg-hangouts-chat-notifications/new-webhook.png") }}

You'll see a long URL displayed representing the API endpoint we can use to send
messages to this room. Copy that URL and save it for later.

{{ figure(src="/img/autopkg-hangouts-chat-notifications/webhook-url.png") }}

## Which postprocessor should I use?

Whichever you like! 

`HangoutsChatNotifier` sends simple messages to inform you when your autopkg
workflow has downloaded new software. The quick notifications might be useful to
send to your desktop support team Hangouts Chat room, or your QA group.

If you're using `JSSImporter` as part of your autopkg workflow, you may find the
expanded detail provided by `HangoutsChatJSSNotifier` useful. Setting up
`JSSImporter` is beyond the scope of this article – but if you're already there,
`HangoutsChatJSSNotifier` will help you send notifications to Hangouts Chat.

## Running the postprocessor

You have two options to include the Hangouts Chat postprocessors in your autopkg
runs.

### Using a postprocessor at the command line

The simplest method is to include the `--post` flag when running autopkg at the
command line. 

#### ...for `HangoutsChatNotifier`

Specify the postprocessor name
`com.github.haircut.HangoutsChatNotifier/HangoutsChatNotifier` and provide a
`--key` in the form `--key hangoutschat_webhook_url="<incoming webhook url>"`.
Replace `<incoming webhook url>` with the URL you noted earlier after creating
an incoming webhook in your Hangouts Chat room.

A complete command might look like:

```
autopkg run MyRecipe.recipe \
--post "com.github.haircut.HangoutsChatNotifier/HangoutsChatNotifier" \
--key hangoutschat_webhook_url="https://chat.googleapis.com/v1/spaces/XXX/messages?key=XXX-XXX&token=XXX-XXX"
```

#### ...for `HangoutsChatJSSNotifier`

Just like the simple notifier, add a `--post` flag to your autopkg run and
specify `com.github.haircut.HangoutsChatNotifier/HangoutsChatNotifier`, and a
`--key hangoutschatjss_webhook_url="<incoming webhook url>"`.

This example might look like:

```
autopkg run MyRecipe.recipe \
--post "com.github.haircut.HangoutsChatJSSNotifier/HangoutsChatJSSNotifier" \
--key hangoutschatjss_webhook_url="https://chat.googleapis.com/v1/spaces/XXX/messages?key=XXX-XXX&token=XXX-XXX"
```

#### ...with a recipes text list

If you use a [text file list of multiple
recipes](https://github.com/autopkg/autopkg/wiki/Running-Multiple-Recipes#recipe-lists)
the postprocessors are compatible and will send a Hangouts Chat message after
each recipe runs.

For example:

```
autopkg run --recipe-list /path/to/recipe_list.txt \
--post "com.github.haircut.HangoutsChatJSSNotifier/HangoutsChatJSSNotifier" \
--key hangoutschatjss_webhook_url="https://chat.googleapis.com/v1/spaces/XXX/messages?key=XXX-XXX&token=XXX-XXX"
```


### Using a postprocessor with a recipe property list

My preferred method of running multiple autopkg recipes is using a [property
list
(plist)](https://github.com/autopkg/autopkg/wiki/Running-Multiple-Recipes#plist-recipe-lists).
A plist recipe list lets you set up declarative and robust autopkg
configurations that would otherwise require a messy string of command line
arguments.

To include the Hangouts Chat postprocessor, add it as an array item under the
`postprocessors` key. Then, add your incoming webhook url as a string under a
`hangoutschat_webhook_url` key.

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>recipes</key>
    <array>
        <string>MyRecipe.recipe</string>
        <string>AnotherRecipe.recipe</string>
        <string>RadicalRecipe.recipe</string>
    </array>
    <key>hangoutschat_webhook_url</key>
    <string>https://chat.googleapis.com/v1/spaces/XXX/messages?key=XXX-XXX&amp;token=XXX-XXX</string>
    <key>postprocessors</key>
    <array>
        <string>com.github.haircut.HangoutsChatNotifier/HangoutsChatNotifier</string>
    </array>
</dict>
</plist>
```

{% alert() %}
  <p>
    <strong>Important!</strong> Hangouts Chat incoming webhook URLs
    include an ampersand (<code>&</code>) in the query parameters, which
    is an illegal character within an XML <code>&lt;string&gt;</code>
    element. You <strong>must</strong> replace the ampersand with its
    character entity <code>&amp;amp;</code> when including the incoming
    webhook URL in a property list file – otherwise the postprocessor
    will fail to send a notification.
  </p>
  <p>
    e.g. <code>.../XXX/messages?key=XXX-XXX&amp;amp;token=XXX-XXX</code>
{% end %}

Now you can run your recipe list and send Hangouts Chat notifications using a
much less cumbersome command:

```
autopkg run --recipe-list /path/to/recipe_list.plist
```

## Using both `HangoutsChatNotifier` and `HangoutsChatJSSNotifier` together

The namespace for each postprocessor is distinct, so you _can_ use both
simultaneously during a single autopkg run. For example, you might like to send
your helpdesk a brief notification that new software is available using
`HangoutsChatNotifier`, and your engineering team managing Jamf Pro a more
detailed note showing the changes made to your JSS.

For this example, you'd create one incoming webhook in your "helpdesk" Hangouts
Chat room and set its URL as the `hangoutschat_webhook_url` key. Create a second
incoming webhook in your "Jamf Pro" Hangouts Chat room and set that URL as the
`hangoutschatjss_webhook_url`. Add both postprocessors to your autpkg run –
done!

## Wrapping up

If you have need to customize the formatting or content of the Hangouts Chat
notifications, you can fork my postprocessors found at
[autopkg/haircut-recipes/PostProcessor](https://github.com/autopkg/haircut-recipes/tree/master/PostProcessors).
Google has [great
documentation](https://developers.google.com/hangouts/chat/reference/message-formats/)
around their message formats.

Now go forth and notify your teams!
