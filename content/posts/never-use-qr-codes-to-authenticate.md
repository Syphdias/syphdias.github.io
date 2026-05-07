---
title: "Never Use QR Codes to Authenticate"
date: 2026-05-07T15:45:01+02:00
---

It was quite late and I was about to logoff to go to sleep, when I got a message
from a friend on Steam. "Hey could you vote for my team in this competition, it
is the $TEAMNAME. Just takes a minute. $URL".

Sure no problem. I haven't seen this in a while, but a decade or two ago (ouch)
this was quite common. So I go to the website — nothing out of the ordinary —
just like in _the old days_, I'd say. So I scroll down to their team and click
on the vote button and it wants to verify that I am a genuine person by
authenticating with my Steam account. Probably some oauth to Steam that just
wants to get basic profile information that my account was not just created two
minutes ago. If it wants too much access I can still deny.

Oh great I can use the QR code to log in. So I pull out my phone and scan the QR
code with the Steam Authenticator app. Vote confirmed, I go to bed.

**This was a mistake** as I will find out after I get up in the morning to
multiple people messaging me: "yo, check your Steam account, I think your
account got hijacked". (Actually most people say "hacked" but that is
technically not the right term.)

I spent the rest of the morning changing the password, checking tokens, etc. to
get the malicious party out of my account and message my entire friend list, to
warn them as early as possible not to click on the link that now I was sending
to people.

I later discovered that the hijacker also removed and blocked some of my friends
because they recognized the phishing attempt and reacted appropriately, such as:
"Could you send me a message on Discord" (reply was Cyrillic) or "Done" (without
actually doing it, which the phisher recognized and got annoyed with "I don't
see your vote").

This experience sucked but in the end it was just my Steam account that I could
easily get back with some annoyances. I am glad it happened to my Steam account
and not my bank account. Because I learned my lesson to never use QR codes to
authenticate.

## How Did This Happen?

So why did this happen? And what could have prevented this?

I cannot say for certain that the following is what has happened. Since at the
time I wanted to check, the website was already offline again.

When I got prompted to log into the fake-Steam I did not check what the URL
said; I assume it was a Steam lookalike website — and to be honest I could not
even say for certain what the real Steam URL is (Steampowered-something dot
com?).

Had I used my password manager, it would not have recognized the website and not
offered to inject my username and password. This would have protected me.

But since I used the QR code to login, there was nothing to check the origin of
the QR code. I assume the website got a legitimate QR to login from Steam web
login and displayed it. When I scanned it and logged myself in, I confirmed the
web login in the background. Note, that I did not enter my password and I did
not "save" the login. This way the attacker only got one session token.

This can happen with anything that allows for QR authentication.

- Step 1: Go to a legitimate website with this mechanism (for example my bank).
- Step 2: Get the QR code(s) and put them into some sort of middleware, where
  the victim might use it.
- Step 3: Get the victim to scan the QR code(s).
- Step 4: The middleware now has access to the real website/app and can do its
  evil deeds. As cover, it just needs to present some sort of success message
  for the victim not to suspect anything.

Obviously the attacker does not do this manually but writes their program to do
this automatically, refresh QR codes, and do the evil bidding if successful.

I say QR codes because my bank requires two consecutive scans of QR codes to log
in but that does not change the scenario, just makes it more involved to write
the middleware.

## The Future of Passkeys

Passkeys have many Pros and Cons which I do not want to go into in detail here.
The Cons unfortunately do not make them a no-brain replacement for passwords
(portability, backup, etc.) but what they do prevent is this kind of attack.

If you create a passkey with WebAuthn for a website, the process includes the
website in the authentication process. The way it works makes it impossible to
use the wrong passkey on the wrong website. It also makes it impossible for a
website's name to change while retaining passkey login (another downside). (This
assumes you do not have a compromised system or browser that does some extra
steps to fake the website's name, etc.)

## Sending Emails

Some websites like Slack allow you to not even set up a password. Instead you
enter your email address and then receive a short one-time pin

This would not protect against this attack. You would enter the email address
into the malicious middleware, get a legitimate email with a pin, and enter the
pin. Pwned.

## What about MFA

Multifactor Authentication will not help in this scenario either as it is also
not tied to a specific domain. I include all MFA here from simple TOTP
(time-based), over HOTP (event-based, like clicking a button to "further the
ratchet"), to specific apps that require you to enter some digits or press
approve (like Microsoft's Authenticator). All these can easily just be forwarded
by the malicious middleware to send it to the legitimate service.

The benefit is that you have to pause and get out another device, hopefully, to
recognize something is off.

## So What To Do?

Use a good password manager that detects the website you are on! Or use a
passkey if you dare. Or store the passkey in your password manager.

**Do. Not. Use. QR. Codes. To. Authenticate.**
