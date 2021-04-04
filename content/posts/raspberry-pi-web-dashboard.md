---
title: "Set Up A Raspberry Pi As An Auto-refreshing Web Dashboard"
date: 2021-04-03T01:00:00Z
tags: ["raspberry-pi", "how-to", "nodejs"]
---

# Set up a Raspberry Pi to run Chromium in kiosk mode

_credit: This section is mostly based on
https://die-antwort.eu/techblog/2017-12-setup-raspberry-pi-for-kiosk-mode/
with some tweak to work with the auto-refresh section later._

## Set up

First, start with a fresh install of
[Raspbian Lite](https://www.raspberrypi.org/software/operating-systems/) -
we don't want all the bloat because we are only using the Raspberry
Pi to display a dashboard

Then, we perform the usual Raspberry Pi setups, including things
like updating password or enabling SSH.

Importantly, run `sudo raspi-config`, in "Boot Options", Select
"Desktop / CLI" and then "Console Autologin". We will auto-run
`startx` later in bash.

Now, install the dependencies - X server, a window manager (we use
the lightweight [openbox](http://openbox.org/wiki/Main_Page)),
Chromium and NodeJS.

```bash
$ curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
$ sudo apt install --no-install-recommends xserver-xorg x11-xserver-utils xinit openbox chromium-browser nodejs
```

A quick note here about `apt` - if you are running commands directly
in your terminal, `apt` offers
[a much better experience](<https://askubuntu.com/questions/445384/what-is-the-difference-between-apt-and-apt-get#:~:text=apt%2Dget%20may%20be%20considered,apt%2Dget(8).>)
than `apt-get`!

## Configuration

Edit /etc/xdg/openbox/autostart and replace its content with the following:

```bash
# Disable any form of screen saver / screen blanking / power management
xset s off
xset s noblank
xset -dpms
```

We disable screen blanking and power management which doesn't make sense for our dashboard.

Next, we want to automatically start the X server on login. We can
modify the `.bash_profile` to include this

```bash
[[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && startx -- -nocursor
```

Note that we have configure the Pi to log into command line above. Combining with
this change it will cause the Pi to start an empty X session on boot.

Now that we are done, we will implement a service to (re)start Chromium on demand.

# Create a service to (re)start Chromium

In my experience, running Chromium for a prolonged period of time
usually results in the classic 'aw snap' crash. A solution here is
to restart the browser from time to time, usually at night when the
dashboard is not needed.

Also, to iterate changes on the dashboard, it's better to have some
mechanism to restart the browser without having to attach a keyboard
to the dashboard display.

Lastly, when we push a new change to the Pi, it's possible that
new code changes can crash the dashboard page, so, we want a way
to refresh that doesn't rely on logics in the web page.

To achieve these, we can implement a service in node.js that
provides an end point to restart the Chromium browser. In addition,
we will launch Chromium in the 'kiosk mode':

> The kiosk mode allows chromium to take over the whole screen with
> only the webpage and no frames or browsers UIs.

Here's a simple, one-file service in TypeScript with minimum dependencies:

```typescript
import { ChildProcess, spawn } from "child_process";
import * as http from "http";

// A global state to keep track of the current chromium process
let curProcess: ChildProcess | null = null;

function stop() {
  if (curProcess !== null) {
    process.kill(-curProcess.pid);
  }
  curProcess = null;
}

function restart(url: string) {
  stop();
  curProcess = spawn(
    "chromium-browser",
    [
      // This disables the annoying 'chrome is out of date' window
      "--check-for-update-interval=604800",
      "--kiosk",
      "--incognito",
      url,
    ],
    { detached: true }
  );
}

// Not the prettiest HTTP service implementation...
http
  .createServer(function (request, response) {
    function fail(error?: Error) {
      response.writeHead(400, { "Content-Type": "application/json" });
      response.end(JSON.stringify({ success: false, error }));
    }

    function succeed() {
      const result = { success: true };
      response.writeHead(200, { "Content-Type": "application/json" });
      response.end(JSON.stringify(result), "utf-8");
    }

    if (request.method !== "POST") {
      fail();
    }

    const restartMatch = /\/restart\??(.*)/.exec(request.url!);
    if (restartMatch) {
      const url = unescape(restartMatch[1]);
      restart(url);
      succeed();
    } else {
      fail();
      return;
    }
  })
  .listen(3001);
console.log("Server running at http://127.0.0.1:3001/");
```

You can also use the [JavaScript code](https://www.typescriptlang.org/play?#code/JYWwDg9gTgLgBAbzgYQBbADYBMAKUIDGApgM4kA0cJYAhgO4B2cAvnAGb4hwBEB62AfTD5iZbgG4AUKEiw4AKjg0ScVDBhh2nHmo0TJkjEXgEArlDyFSJAFwp+uEdbgAfOA1MYMcALzvPGFKSbKYMBDDAEEwkMBBgABQAlIiScHDAbHDxZhZOZHAAhD5+Hl7JCKlpcMJWZAB0ANaYGPEAtDmWoiR1YMBYiVJpzJUdeSolAVLDwaHhkUxQpDA0sPHmGHYxUMAMAObllTFxSYNwo7XjVLSM8ZVpvKicwKYgrQBG+HQkRFDc5HdwADaALSAHpQXAACroFRYYAkGhvIwqGCoIhKBgMCAATx2uzgAHI+Jx0fC4BBTPAIJksDQYEQCXA6DssBA6CCeK12miCA1WmxoK1TGBafTWjt6VAAG40DA+ABsAAYACwADkVir+HO4XKaEBIDS1VSqOvFYQguwYwFiRuNcHW-2NAF1HVUkFhjDQ+EQsHYYFBTOjhmkBpJprowJU6gRFnSiABlH5Sn7xEJhCJRLKLACOgZilEW1Ci3wOxrTc0zbBomHiP3wUAA-HYAKJQeulu2FyAMb51OjbekACSINCw8WVGsoSG4yCi9IYMFakOxYCI3Ds3BoYDAGGABDp81BACsSFFuCxQ3a4F3i0Q6kQGGOAFLxgDyADk6ls8RlsfEkCQpgEF0dhVhg3yUHW0AXpeVTTGWswZtEQHED6SQpFeBDFvAhaePAfgASh1h+gGQanMaN49ne-bWkQw6jvEABMk6IDws4Lg+i7Lqu648FuO57geUTHqeDDnswsEUaQ3a9g+z5vp+357L+8S4RgMCJJQ3CUmwrSqtwkksAYxoZFmRC5ksdQgMYqAQFghTFDwOCvvGkIGRhdpVjWhnwVUWE9jhSwrDAACydJ8L4cCggAOqChbLLA0UNg28R1PIiSgveAAeRAEKp5l5jAdTrAUhmmflMTBWFMB8B2xr+TE9pQN4fihKQ+6rhVCWheFqCAgAjE6hlpPFwVrM1w1UERaE+XARDgeiFRXl5LSTYsMDmAw5FGUMiRRruMQPvEADMGr9aGDUQEYdQYBa8TcIm0o-NeoRWnsSjwBGNjgv1jEAOx1IqgN1P1NinYq-WggZ4hAA) directly.

We use an HTTP server, but we can also use websocket or
whatever else that's convenient for the use case.

Lastly, we can set up a new service to make sure that this service
starts running when we reboot the Pi. To do so, we create a new systemd service.
For example, `browser_config.service` in `/etc/systemd/system`:

```toml
[Unit]
Description=BrowserControl

[Service]
# We need this to make sure that chromium is opened in the right X server session
Environment="DISPLAY=:0"
Environment="XAUTHORITY=/home/pi/.Xauthority"
WorkingDirectory=/path/to/compiled/js/
ExecStart=/usr/bin/node /path/to/compiled/js/file_name.js
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=browser-control-service
User=pi
Group=pi

[Install]
WantedBy=graphical.target
```

Then, we make sure to start this service on boot:

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl start browser_control_service
$ sudo systemctl enable browser_control_service
```

Finally, to restart the sole chromium browser, your app or script can post
to `http://localhost:3001/restart`, e.g.

```bash
$ curl -X POST http://localhost:3001/restart?http://example.com
```
