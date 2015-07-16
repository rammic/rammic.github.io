---
layout: post
title: "This is an intervention. Stop storing secrets in your apps."
description: "Mobile developers- Stop storing secrets in your apps."
category: 
tags: []
---
{% include JB/setup %}

This is a repost of a submission I made to Reddit's [/r/androiddev](https://reddit.com/r/androiddev) subreddit. You can [pull up the thread](https://www.reddit.com/r/androiddev/comments/3dedaj/dear_android_developer_this_is_an_intervention/) to see the ensuing discussion.

------

Reading the [comments on the cloned app](https://www.reddit.com/r/androiddev/comments/3dblmo/am_exact_copy_of_my_app_has_been_uploaded_to_the/) post nearly gave me a nervous breakdown. Fellow devs, please sit down and let's have a frank conversation about your android app's security. 

It's time you hear and accept this: **Your app doesn't love you and it will happily betray your trust in the right hands**. Don't believe me? Check out the countless posts from /r/netsec, /r/reverseengineering, /r/pwned [about breaking into android apps](https://www.reddit.com/r/reverseengineering+netsec+pwned/search?q=Android&amp;restrict_sr=on&amp;sort=relevance&amp;t=all). A skilled RE (Reverse Engineering) hacker will quickly convince your app to give up all of its closely guarded secrets and do all of those nasty little actions that you took such great pains (and unit tests) to prevent.

Now I know this is hard to hear and I can already anticipate what you're going to say:

* **But hacking is impossible without access to my apk!** Getting access to an apk is simple. Once downloaded to a rooted phone, pulling it via ADB is one *adb pull /data/app/[your_apk_name]* command away. You can even download an APK from Google Play [with the right help](https://chrome.google.com/webstore/detail/apk-downloader/cgihflhdpokeobcfimliamffejfnmfii). 
* **But I used Proguard on my code!** Great! Did you include any sensitive URLs, strings, keys, passwords, or other resources in your app? [Then you're still screwed](http://proguard.sourceforge.net/FAQ.html#encrypt). Proguard only makes your app's code ["harder to reverse engineer"](https://developer.android.com/tools/help/proguard.html)- extracting all of those juicy unprotected strings and resources from your code is still insanely easy. Even modifying a proguard-protected app to bypass a boolean security check is the first thing that you learn to do [when patching an app](http://leonjza.github.io/blog/2015/02/09/no-more-jailbreak-detection-an-adventure-into-android-reversing-and-smali-patching/).
* **But I cleverly disguised and encrypted my strings/keys/urls!** Do you ever communicate those back to a server? There are [plenty of tools](http://www.charlesproxy.com/) that will readily intercept and display any network communication and endpoints- yes, even if it's SSL/TLS protected. Even if you were careful to never reveal them on the wire, it's still possible to yank the sensitive bits directly from the memory a running application.
* **But I implemented certificate pinning!** That's a good thing for your users' security, but it give you no assurances regarding the trustworthiness of your app. Certificate pinning [is still easily bypassed if someone intends to do so](https://www.reddit.com/r/netsec/comments/3csh17/reverse_engineering_the_subway_android_app/).

Bottom line: Just like with testing, you should **never assume that your mobile app will only operate the way you originally intended it to**. Ever. Here are the harsh realities of android development (or any mobile app for that matter):

* **How do I prevent someone from subverting/cloning my app?** You can't. You can make it harder through obfuscation, indirection, etc. In most cases it'll only provide an annoyance and delay to a determined reverse engineer- a scenario that you absolutely will encounter if your app is at all successful. Also remember that each deterrent you bake makes your app more difficult to manage and is another opportunity to [anger an otherwise legitimate user](https://en.wikipedia.org/wiki/Digital_rights_management) when something goes wrong. Even for companies like Google that do prevent most client-side exploitations, it's a never ending, incredibly expensive arms race between their vast security teams and individual hackers.
* **How do I perform business-sensitive operations then?** If you need to do something that you must be absolutely sure is not subverted, only do them on a server you own and control. This is especially true with any financial transactions, including [verification of in-app purchases](https://developer.android.com/google/play/developer-api.html).
* **But I don't have the money for a server.** Stand up a free AWS instance, heroku dyno, or a $5/month DigitalOcean droplet. If you're serious enough about your app to try and make money from it, keeping your most sensitive operations on systems you own and control is a simple and cheap safeguard.
* **This all sounds a bit nihilistic. Why do anything then?** Realize that there's two different types of security you should be considering- *User Security* and *Business Security*. You should be focused on user security with your mobile apps- it's about making sure that your users don't accidentally do anything that would negatively impact them. Things like cert pinning, TLS/SSL, verifying inputs, etc. are all important for user security. But, just like in life, you can't protect people from hurting themselves or doing bad things. So for your own security, you must always assume that you have malicious users out there (i.e. users of your mobile app) that will lie, cheat, and otherwise ignore all of those virtuous safeguards you put in place for them. 

TL;DR- Your app is not your trusted friend. Never included anything in it (e.g. passwords, secret URLs, API keys, etc) that you wouldn't want someone to see. Securing your app should be about protecting your users' security and privacy- any sensitive operations or information that you or your business depend on being secret or done absolutely correctly should be handled exclusively on systems you own and control.