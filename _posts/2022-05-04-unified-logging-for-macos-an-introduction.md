---
layout: post
title: "Unified Logging for macOS, an introduction."
date: 2022-05-04
slug: unified-logging-for-macos-an-introduction
---

**`*What does the logs says?*`

I couldn't think of any different introduction to approach Unified Logging rather than this statement that is my personal mantra of troubleshooting/looking into any issue when it comes to end user devices.
Logging is the unique base for any device related issue for sure. 
But it can be much more, offering an insight and overview of apps, technologies and products running on our devices.

`Where to start?`

Traditionally, macOS and any 3rd party have logged into the "standard" location of `/var/log `, de facto writing out logs into files on disk.

Apple introduced radical changes in with macOS 10.12.x where Apple moved the system and process logs to a unified database on the Mac.


Let's start with some references:


- Apple Unified Logging Documentation - [https://developer.apple.com/documentation/os/logging](https://developer.apple.com/documentation/os/logging)
- Apple WWDC 2016 Into to Unified Logging - [https://developer.apple.com/videos/play/wwdc2016/721/](https://developer.apple.com/videos/play/wwdc2016/721/)
- "Unified Logging and Activity Tracing" - WWDC 2016, Session 721 - [https://devstreaming-cdn.apple.com/videos/wwdc/2016/721wh2etddp4ghxhpcg/721/721_unified_logging_and_activity_tracing.pdf](https://devstreaming-cdn.apple.com/videos/wwdc/2016/721wh2etddp4ghxhpcg/721/721_unified_logging_and_activity_tracing.pdf)


The last one is a very comprehensive session, lets try to over-simplify the content and try to describe what Unified Logging is, in a nutshell:


- Single efficient logging mechanism, offering a comprehensive and performant API
- Maximize information collection with minimum observer effect
- Compression of log data
- Managed log message lifecycle more than just time stamps
- a common system across macOS, iOS, watchOS, tvOS
- Designed with Privacy in mind and a Privacy first mindset


 
Now that we have a bit of background of what Unified Logging is (UL for brevity in this post), let's dig a bit more into.

Logs are stored in the `tracev3` formatted files in `/var/db/diagnostics`, which is a compressed binary format. 
As with all binary files, you’ll need new tools to read the files. The Console.app is a great UI to explore the logging, but it can be quite overwhelming. 


To get our hands on logs, we can use the `log` command, on macOS - `/usr/bin/log`


To get more information, simply open the Terminal.app and run:
`man log`


The `log` command can be used to Collect and View Unified Logging data and.
There are 3 main verbs for the `log` command that is particularly interesting to look at:


- `log collect --output path` this allow us to generate a .logarchive file
- `log show `this allow us to have macOS produce a log file of everything we've specified in our filter search, limited by time using the `--last` flag, for example if we're interested only on the last 60 minutes of logging we will set `--last 1h`
- `log stream` this is a live streaming of everything we've specified in our filter search, it will not show any historical data, only starting from when the command is executed.


Ok, so that's it? Simple, we can now just go in Terminal.app and run `log show`!
Yes, but no. If you try that, you'll see how comprehensive and extensive UL is, you will basically be flooded by information.
We've seen above we can use the `--last` flag to time-restrict the search and limit the logging to a specific period of time (we can go as low as 1 minute).
But that is not sufficient, given the verbosity of the output.


Here in rescue comes Predicates**!
As usual, lets first start with the docs: [https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/Articles/pSyntax.html](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/Articles/pSyntax.html)
Predicates are an option that provide granular control over filtering and provide the logic to confine a search to specific values.
Predicates gets hooked up in the `log` command query filter:


`log steam --predicate '(predicate_here)'`

For example, we can filter looking at data from Safari:


`log stream --predicate '(process=="Safari")'`


Predicates available:


```
 eventType The type of event: activityCreateEvent, activityTransitionEvent, logEvent, signpostEvent, stateEvent, timesyncEvent, traceEvent and
 userActionEvent.

 eventMessage The pattern within the message text, or activity name of a log/trace entry.

 messageType For logEvent and traceEvent, the type of the message itself: default, info, debug, error or fault.

 process The name of the process the originated the event.

 processImagePath The full path of the process that originated the event.

 sender The name of the library, framework, kernel extension, or mach-o image, that originated the event.

 senderImagePath The full path of the library, framework, kernel extension, or mach-o image, that originated the event.

 subsystem The subsystem used to log an event. Only works with log messages generated with os_log(3) APIs.

 category The category used to log an event. Only works with log messages generated with os_log(3) APIs. When category is used, the subsystem
 filter should also be provided.
```


Now the last one in the list, `subsystem`, is a very interesting one as it becomes very helpful to narrow down the search filter.
Charles Edge [krypted](https://twitter.com/cedge318) did a great job providing a list of Apple's `subsystems`: https://gist.github.com/krypted/495e48a995b2c08d25dc4f67358d1983


For any other .app on a device, we can pull its subsystem from the Info.plist using a command like the below:


`defaults read /Applications/APP_NAME.app/Contents/Info.plist CFBundleIdentifier`


Example with Jamf Protect:


`$ defaults read /Applications/JamfProtect.app/Contents/Info.plist CFBundleIdentifier
com.jamf.protect.daemon`


In this example, if we would like to stream Jamf Protect data logged, a very simple UL search could be like:


log stream --predicate "subsystem == 'com.jamf.protect.daemon'


Some examples from Apple's man page:


```
Filter for specific subsystem:
log show --predicate 'subsystem == "com.example.my_subsystem"' 


Filter for specific subsystem and category:
 log show --predicate '(subsystem == "com.example.my_subsystem") && (category == "desired_category")'


 Filter for specific subsystem and categories:
 log show --predicate '(subsystem == "com.example.my_subsystem") && (category IN { "category1", "category2" })'


 Filter for a specific subsystem and sender(s):
 log show --predicate '(subsystem == "com.example.my_subsystem") && ((senderImagePath ENDSWITH "mybinary") || (senderImagePath ENDSWITH "myframework"))'
```


Log verbosity.
There are different level of logging verbosity available:


- Default
- Info
- Debug
- Error
- Fault


Unless specified, the Default logging level is always the default (who would have guessed) level.
This can be configured in the UL filter:

`log (stream/show) --predicate (--debug/ --info)`


In some specific scenarios, it is useful to enforce the default logging of a binary (or an app) to debug for troubleshooting purposes.
To ensure debug level logging is saved for a given subsystem you can run one of these commands:


`log config --subsystem "com.jamf.management.daemon"--mode "level:debug,persist:debug"`

`log config --subsystem "com.jamf.management.binary"--mode "level:debug,persist:debug" `


Make sure to reset to default settings once done:


`log config --subsystem "com.jamf.management.daemon"` `--reset`

`log config --subsystem "com.jamf.management.binary"` `--reset" `


Logging levels can also be deployed via MDM: [https://developer.apple.com/documentation/devicemanagement/systemlogging](https://developer.apple.com/documentation/devicemanagement/systemlogging)


```
<plist version=”1.0”>
<dict>
 <key>PayloadContent</key>
 <array>
 <dict>
 <key>Processes</key>
 <dict/>
 <key>Subsystems</key>
 <dict>
 <key>com.example.app</key>
 <dict>
 <key>DEFAULT-OPTIONS</key>
 <dict>
 <key>Level</key>
 <dict>
 <key>Enable</key>
 <string>Info</string>
 </dict>
```


Now that we have explored a bit how Unified Logging is built, what's the data structure and interactions, let's see some Jamf related examples of UL filters:


We can start with a very basic streaming for the management framework using either the `info` or `debug` log verbosity level:


`/usr/bin/log stream --predicate 'subsystem BEGINSWITH "com.jamf.management"' --level info`

`/usr/bin/log stream --predicate 'subsystem BEGINSWITH "com.jamf.management"' --level debug`


Or, for example, we can look at specific days only and restrict our search to those only - for example last month of logs - date format YYYY-MM-DD:


`/usr/bin/log show --predicate 'subsystem BEGINSWITH "com.jamf"' --style compact --debug --start '2022-04-01' --end '2022-05-01'`


Or we can also format for a specific time range: YYYY-MM-DD HH:MM:SS:


`/usr/bin/log show --predicate 'subsystem BEGINSWITH "com.jamf"'--style compact --debug --start '2022-05-02 10:00:00'--end '2022-05-02 11:00:00'`


Last mention on privacy. As we saw, Unified Logging is designed to not log any kind of private information.
For investigation purposes, this might be disabled. 
Within the [SystemLogging](https://developer.apple.com/documentation/devicemanagement/systemlogging) documentation, there is mention of a specific key that can enable the logging of Private Data:


```
 <key>Enable-Private-Data</key>
 <true/>
```


As an example, we can look at some Safari UL logging obtained with a filter like the below:


`log stream --predicate '(process == "Safari" and message contains "Look up" || message contains "databases")' --debug`


Without `Enable-Private-Data` set to true (or with the key pair set to false), when browsing to a website URL within Safari, the actual URL is masked behind a `<private>`


` Debug 0x0 592 0 Safari: (SafariSafeBrowsing) [com.apple.Safari.SafeBrowsing:Lookup] Look up a ur`l <private>


When `Enable-Private-Data` is enabled, with the same UL filter we can actually see the ULR that has been browsed:


` Debug 0x0 592 0 Safari: (SafariSafeBrowsing) [com.apple.Safari.SafeBrowsing:Lookup] Look up a url https://www.google.com/`


Keep in mind that Private-Data is completely transparent to the end-user if it’s enabled.
If should only be enabled for specific troubleshooting purposes and disabled immediately after testing is completed.


As we've seen, Unified Logging is extremely powerful and can be very very flexible.
If offers virtually unlimited customization.


A very good example of usage can be found here [https://www.modtitan.com/2022/03/in-weeds-with-app-installers-preview.html](https://www.modtitan.com/2022/03/in-weeds-with-app-installers-preview.html) where [Dr. K](https://twitter.com/emilyooo) brilliantly debugged Jamf's App Installers using Unified Logging filters.


What usage can be made of UL then?


For instance, an example is provided by Jamf Protect, which can pull specific UL filtered logs and ship them onto a SIEM of your choice.
There's quite a few UL filters example on the Protect GitHub repository: [https://github.com/jamf/jamfprotect/tree/main/unified_log_filters](https://github.com/jamf/jamfprotect/tree/main/unified_log_filters)
