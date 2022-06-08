+++
title = "How to Parse JSON on the macOS Command Line Without External Tools Using JavaScript for Automation"
date = 2021-11-14
updated = 2021-11-24
path = "parse-json-command-line-mac"
aliases = [
    "posts/how-to-parse-json-macos-command-line"
]
description = "One way to parse JSON on the macOS command line, using only native tooling without external dependencies like jq or similar."
[extra]
author = "Matthew Warren"
+++

JSON – [JavaScript Object Notation][jsonwiki] – is the lingua franca for
shipping data between systems. Everything from software APIs to web services
commonly support, and typically default to, outputting data in JSON format.

Because of its ubiquity, you're bound to run into a need to manipulate a chunk
of JSON in the course of managing your fleet.

For example, you might run a shell script on your Macs that instructs them to
read data from an external system via its API using `curl`. That external system
returns a big ol' heap of JSON, but you need to extract only a single data point
from that payload, then act on that value.

This is somewhat of a challenge in the shell because macOS does not include any
native or pre-installed tools to help you hack down that JSON. Administrators
often resort to workarounds like shelling out to Python or using `sed` and `awk`
to approximate parsing.

However, another, **native** solution is right there in the name: use
JavaScript.

<!-- more -->

### JavaScript, JXA and Open Scripting Architecture

It seems appropriate to use JavaScript to parse _JavaScript_ Object Notation,
right? But how? Can you do that without relying on external tools?

Since Mac OS X Yosemite, Apple has supported JavaScript as a language to
interact with the [Open Scripting Architecture (OSA)][osa]. OSA enables
interapplication communication for automation, and using JavaScript for this
purpose is known as _JavaScript for Automation_ (JXA).

JXA is built right into macOS and lets you use JavaScript without any
additional tooling.

You may be familiar with using the `osascript` binary to execute AppleScript
snippets. While AppleScript is a more common choice for accessing OSA
functionality, you can trivially use JavaScript instead.

On the command line, use the `-l` (that's a lowercase L) flag  with `osascript`
to specify `JavaScript` as the scripting language. Without the flag, `osascript`
defaults to expecting AppleScript. 

Here's a simple example derived from [Apple's sample code][applesample]:

```javascript
osascript -l 'JavaScript' -e 'var app = Application.currentApplication(); app.includeStandardAdditions = true; app.displayDialog("Hello from JavaScript!");'
```

This will display a dialog window using JavaScript. Easy!

### Using JXA to read JSON in a shell script

Let's look at an example where we use the [ip-api.com][ipapi] service to query
the current geographic location of the system based on its IP address.

```javascript
#!/bin/bash

GEODATA=$( curl -s http://ip-api.com/json/ )

read -r -d '' JXA <<EOF
function run() {
	var ipinfo = JSON.parse(\`$GEODATA\`);
	return ipinfo.city;
}
EOF

CITY=$( osascript -l 'JavaScript' <<< "${JXA}" )

echo "This computer is in ${CITY}"
```

First, we call the ip-api service by using `curl` to send a `GET` request to the
service's JSON endpoint. The results are stored in a variable called `GEODATA`
via [command substitution][comsub].

Next, we use the `read -r -d '' VARIABLE_NAME <<LIMIT_STRING` "bashism" to store
a multi-line JavaScript program as a variable using [here-document][heredoc]
notation. This makes it easier to write and maintain the JavaScript code, rather
than trying to smoosh it into a single line.

This (extremely basic) JavaScript program takes advantage of JXA's special
`run()` function. `run()` is automatically executed when the program is called,
and will print any output returned.

Within the JavaScript code, notice the assignment of `ipinfo` references the
`GEODATA` variable defined in the outer shell script via variable expansion. The
JSON returned by ip-api.com is a native JavaScript data structure that could be
used directly, but for safety we treat it as a string and parse it using the
[`JSON.parse()`][jsonparse] method. Since most JSON payloads are multi-line
strings, we wrap the variable expansion in <code>``</code> backticks to take
advantage of [JavaScript's template literal][templit] feature. And finally,
those backticks need to be escaped using `\` backslashes since they're embedded
within a shall script. Whew!

{% alert() %}
<p>When I first published this article, I advised that you could reference and use
a JSON object received from a remote source directly, without running it through
<code>JSON.parse()</code>.</p>

<p>This is true, but it's not secure. I was assuming the safety of the external
data.</p>

<p><a href="https://paulgalow.com/how-to-work-with-json-api-data-in-macos-shell-scripts" title="How to work with JSON API data in macOS shell scripts without external dependencies">Paul Galow wisely pointed out</a> that a malicious source could
potentially send JavaScript instead of JSON, which our script would dutifully
execute. RCEs are bad. Be like Paul and run payloads through
<code>JSON.parse()</code>. This ensures things that aren't JSON cause the script
to error.</p>
{% end %}

A quick aside on quoting and here-documents: Quoting the limit string (`EOF` in
this case) will **prevent** variable expansion within the here-document.
`<<'EOF'` would cause `$GEODATA` to be interpretted literally rather than
replaced by the results of the `curl` command. The unquoted `<<EOF` allows the
variable to expand as expected.

Next, we simply return the `city` attribute of the `ipinfo` object.

Finally, we use the `osascript` binary to run the JavaScript program.
`osascript` can read input from `stdin`, so we use `<<<` to read the program
stored in the `JXA` variable. Storing its output in the `CITY` variable lets us
use it elsewhere in the shell script. In this case we're outputting a nice
message. Running the script will print `This computer is in Atlanta` (or
your approximate location) to your terminal.

That was an extremely verbose explanation of a very basic program! But,
hopefully it helps explain the patterns being used here. Now let's try something
more complex.
### Complex JSON manipulation

Our next example will demonstrate the power of JavaScript's native JSON
capabilities. It's a neat trick to pull a single attribute out of a simple JSON
payload – but it's **actually useful** to manipulate the data using
conditional logic.

Here we'll pull the recent commit history of the AutoPkg GitHub repository, then
retrieve the commits whose commit message length is longer than the soft
50-character recommended maximum length.

Here's the code:

```javascript
#!/bin/bash

GITHUB=$( curl -s https://api.github.com/repos/autopkg/autopkg/commits )

read -r -d '' GITCOMMITS <<EOF
function run() {
	const commits = JSON.parse(\`$GITHUB\`);
	const long_messages = [];
	for (const commit of commits) {
		if (commit.commit.message.length > 50) {
			var msg = "'" + commit.commit.message + "' - Commit " + commit.sha;
			long_messages.push(msg);
		}
	}
	return long_messages.join("\n");
}
EOF

osascript -l "JavaScript" <<< "${GITCOMMITS}"
```

The basic structure is largely the same as in the previous example: use `curl`
to gather data, then feed that into a JavaScript program.

The cool part is being able to manipulate the JSON data in place. Here, we're
iterating through each commit and checking the string length of each commit
message. If that length is longer than 50 characters, we append that message to
an array, along with the commit's SHA hash. Finally, we return a newline-joined
string of those commits.

The end result prints only those commit messages longer than 50 characters.

This example would have been _extremely_ cumbersome to implement in pure shell.
With JXA, it's just some relatively simple JavaScript.

### Reading JSON files from disk

You can also read JSON files that exist on disk with just a tiny bit of extra
effort.

```javascript
#!/bin/bash

read -r -d '' PARSE <<EOF
var app = Application.currentApplication();
app.includeStandardAdditions = true;

function parseJSON(path) {
    const data = app.read(path);
    return JSON.parse(data);
}

function run() {
    const parsed = parseJSON("/path/to/some/file.json");
    return parsed.some.key;
}

EOF
```

This time, we're using using JXA to get an instance of the "current
application." This lets us do useful things like access the file system. We
define a reusable function to parse a JSON file at a path supplied as a
parameter. Within the `run()` function, we call that `parseJSON()` function and
supply the full path to a JSON file. We can then access the parsed JSON
properties using standard [dot notation][dotnotation].

### Other tools and alternatives

Why go through the trouble of using JXA for these use cases?

First, it's really not much "trouble." I'd be implementing similar logic in
another external language or tool. This is more or less adding a thin wrapper.
As long as you are comfortable writing JavaScript (or searching StackOverflow),
it's very little extra effort.

But why not use another tool or method?

When running code on client systems within a fleet, it's best to use the tools
those clients have available by default.

Since [Apple will soon stop shipping scripting runtimes][deprecated] such as
Python, Perl, and Ruby, you **should not** rely on "shelling out" to those
languages. Sure, it might be easy enough to add a `parse=$( python -c 'import
json;...' )` call to your script; but at this point you'd just be creating new
technical debt. Don't do that.

If you ship your own Python 3 runtime – such as with "[macadmin
python][macadminpy]" – you could, of course, use that. Installing your own
Python 3 framework on your clients is a great idea if you routinely use Python
code to manage them. Otherwise, it _is_ another external dependency.

Alternatively, you _could_ pull down [jq][jq] on your client systems. jq is
built specifically to handle JSON on the command line and in shell scripts. But
again – it's an external dependency.

Using JXA requires no new installations or configuration changes. It will simply
run on your systems as they currently exist.

For that reason, I think JXA a great tool to reach for when you need to work
with JSON.

Plus, constraints are fun, right?!

### Further reading

Interacting with JSON is a common need, but these examples barely scratch the
surface of what you can achieve with JavaScript for Automation.

JXA even includes an [Objective-C bridge][objc], which allows you to use macOS
frameworks like AppKit and Foundation.

The [JXA Cookbook][jxacookbook] on GitHub contains a wealth of information and
is an essential read on this topic. It's a great supplement to Apple's somewhat
[sparse documentation][applejxa].

[jsonwiki]: <https://en.wikipedia.org/wiki/JSON>
[osa]: <https://developer.apple.com/library/archive/documentation/LanguagesUtilities/Conceptual/MacAutomationScriptingGuide/HowMacScriptingWorks.html>
[applesample]: <https://developer.apple.com/library/archive/documentation/LanguagesUtilities/Conceptual/MacAutomationScriptingGuide/DisplayDialogsandAlerts.html#//apple_ref/doc/uid/TP40016239-CH15-SW1>
[ipapi]: <https://ip-api.com>
[comsub]: <https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html>
[heredoc]: <https://tldp.org/LDP/abs/html/here-docs.html>
[dotnotation]: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Property_accessors>
[deprecated]: <https://developer.apple.com/documentation/macos-release-notes/macos-catalina-10_15-release-notes#:~:text=49764202>
[macadminpy]: <https://github.com/macadmins/python>
[jq]: <https://stedolan.github.io/jq/>
[jxacookbook]: <https://github.com/JXA-Cookbook/JXA-Cookbook>
[applejxa]: <https://developer.apple.com/library/archive/releasenotes/InterapplicationCommunication/RN-JavaScriptForAutomation/index.html>
[objc]: <https://github.com/JXA-Cookbook/JXA-Cookbook/wiki/Using-Objective-C-%28ObjC%29-with-JXA>
[paulgalow]: <https://paulgalow.com/how-to-work-with-json-api-data-in-macos-shell-scripts>
[templit]: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals>
[jsonparse]: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse>
