# svsc unattended install documentation

This document serves as a reference for myself and others who are installing a digital art piece on a machine that will be unattended for an extended period of time. 

> _Wait, arent there tons of guides for that already?_

Why yes, yes there are! But I wanted to document the litle things that I needed to do which might have not been covered in those guides or that might have changed due to newer hardware, operating systems, or any other reason.

The main guide I followed was https://github.com/laserpilot/Installation_Up_4evr

This guide is for installations on macos based hardware, but has links for [Ubuntu](https://github.com/brangerbriz/up-4evr-ubuntu) and [Windows](https://github.com/brangerbriz/up-4evr-ubuntu)

## requirements

The piece I was working to show was a generative audiovisual artwork "serendipity vs consequence". Without going into much detail, the code in the work renders differently depending on the graphic engine used in the machine and operating system running the code. The code is the same, there is no conditional logic which is triggered when different graphic engines are detected. This being said, the piece looks great on computers running Metal, Apple's graphic engine for M chips. 

The code should run locally to avoid any possible wifi or network connection issues.

This establishes the first requirement: a machine with an M chip running macos.

This machine should be able to start up on its own and start all of the software necessary to get the pice shown. As a good measure, the machine should turn off at night and turn on in the morning by itself.

When the code for svsc was written, there were some things I was doing with three js post processing that were not supported on chrome, so the piece should run on firefox in the installation.

The work should be shown full screen and with an activated audio context, things which typically require user interaction to activate on a browser.

Given these requirements, I will now detail how I set up the hardware and software.

### hardware and operating system

I chose a Mac Mini M1 with macos 14.x for the installation. This means that the guide I posted above is already a bit out of date, since it was written in 2012. Features have ben dropped, moved, or obscured in the macos ui. The guide was still very very helpful to know the important settings to tweak, and when I could not find them, I would verify their removal or find where they were. If the feature was removed, I found a workaround.

I followed the ["Prepare the Computer"](https://github.com/laserpilot/Installation_Up_4evr?tab=readme-ov-file#prepare-the-computer) section as closely as possible to ensure that any notifications were suppressed

I wanted to schedule the machine to turn off and on every day, and this seems to have been removed from macos 14. So instead I used [Onyx](https://www.titanium-software.fr/en/onyx.html)

### software

The main issue to resolve was how to start up the artwork when the computer started up. For this I went with creating a .plist file that ran a shell script. 

The plist file looks like this:

````xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
   <key>Label</key>
   <string>com.user.loginscript</string>
   <key>ProgramArguments</key>
   <array><string>/Users/svsc/dev/svsc-paloalto/start/start.sh</string></array>
   <key>RunAtLoad</key>
   <true/>
   <key>WorkingDirectory</key>
   <string>/Users/svsc/dev/svsc-paloalto/start/</string>
</dict>
</plist>

````
Setting the working directory was key to making sure the commands in the shell script executed correctly.

The shell script looks like this: 

```bash
#!/bin/bash
export PATH=/usr/local/bin/npm:/usr/local/bin:$PATH
cd /Users/svsc/dev/svsc-paloalto/src && npm run present
```
I export a few paths so that the process can find npm.

I ended up using explicit paths cause I got paranoid, but since the plist has a working directory set, this should not be necessary.

The npm script runs:

```
 "present": "webpack serve --config ./config/webpack.config.present.js"
```

Which starts a webpack server with the following config:

```js
const config = require("./webpack.config")

module.exports = {
  ...config,
  mode: "development",
  devServer: {
    // disables the Hot Module Replacement feature because probably not ideal
    // in the context of generative art
    // https://webpack.js.org/concepts/hot-module-replacement/
    hot: false,
    port: 8080,
    open: ['http://localhost:8080/frame.html'],
    client: {
      overlay: {
        errors: true,
        warnings: false,
      },
    },
  },
}
```

This config is mostly copied from the [fxhash template](https://github.com/fxhash/fxhash-boilerplate).

You see that this opens the served page but I direct it to another html file called frame.html. This is an html file with one fullscreen iframe which has the index.html of the artwork as its source. This helps me ensure that the piece is always fullscreen.

I wanted to show a different iteration of the work every minute, so I did have to add this code to the artwork code itself:

```js
setTimeout(function(){ 
  location.reload();
}, 60000);
```

If I added it into the frame.html it would loose fullscreen. Maybe there are better ways to go about it, but this works.

How did I get firefox to go full screen programatically? I haf to set the `full-screen-api.allow-trusted-requests-only` flag in `about:config` to **false**. 

Note: firefox is set to be the default browser on the system, so when webpack starts to serve, it opens firefox.

The audio context starting automatically was a tricky issue, but I was able to tell firefox to always allow autoplay from localhost:8080

### other considerations

I could not for the life of me get the mouse cursor to hide. My first solution was to move it programatically with AppleScript to the bottom right corner. The script looked like this:

```applescript
use framework "Foundation"
use framework "CoreGraphics"

tell application "Finder"
	set _b to bounds of window of desktop
	set _width to item 3 of _b
	set _height to item 4 of _b
end tell

set cursorPoint to current application's NSMakePoint(_width, _height)

set theError to current application's CGWarpMouseCursorPosition(cursorPoint)

if theError = 0 then
	return "SUCCESS"
else if theError = 1 then
	return "FAILURE"
end if
```

At one point I was running this as part of the startup shell script:

```bash
#!/bin/bash
export PATH=/usr/local/bin/npm:/usr/local/bin:$PATH
osascript /Users/svsc/dev/svsc-paloalto/start/move.scpt & cd /Users/svsc/dev/svsc-paloalto/src && npm run present
```

where `osascript /Users/svsc/dev/svsc-paloalto/start/move.scpt` would execute the script.

A kind soul suggested I try something called [cursorcerer](https://doomlaser.com/cursorcerer-hide-your-cursor-at-will/) out which gives you a small UI to set how you want the mouse to hide or show. It did the trick very nicely and I no longer needed to use the AppleScript.