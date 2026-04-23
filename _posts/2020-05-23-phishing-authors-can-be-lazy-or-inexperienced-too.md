---
layout: post
title: "Phishing authors can be lazy or inexperienced too"
date: 2020-05-23 20:25:00 +0200
categories: [Information Security, Phishing]
tags: [Analysis, Command Injection, Phishing, Vulnerability]
description: "The other day I received a blatantly obvious phishing email and I decided to analyze it and look through the website. There were a few things that really surprised me."
author: Milan Habrcetl
---

## The phishing email

It was Saturday afternoon and I was a little bit bored, so I decided to check the trends on phishing in my email inbox. I came across this "masterpiece" of a phishing email:

![Phishing email]({{site.baseurl}}/assets/2020/05/email.png)

So I had my first laugh with that as it seems that the author did not invest **ANY** time in making of this "masterpiece". For those not versed in spotting phishing emails there are numerous signs in this email that can tip you off, some of them are:

- There does not seem to be any text, just a button.
- There are mistakes in the few words that are present ("Accounte" in the Subject, or the poorly written Czech text on the button).
- The sender is named Root User (what? why?).
- The sender's email is from a Russian domain that does not even remotely look like PayPal's domain (Really? This is just lazy...).

![Dr. Evil Meme - Masterpiece]({{site.baseurl}}/assets/2020/05/masterpiece.png)

Okay, so now we know this is a phishing email. What can we do with it? Normally I would just delete it and move on but as I was bored, I decided to look at it some more.

Firstly I noticed there was a text after all:

![Phishing Email with the text highlighted]({{site.baseurl}}/assets/2020/05/email-highlighted-1024x235.png)

Great, so it seems that the author used some sort of bad translator for the text. Next I hovered over the link to see what website I would be taken to after clicking (this is also a good way to spot phishing emails).

The link would take me to `hxxps://www[.]counterfeitmoneyuk[.]com/Update/sign-in` (the protocol was https, but I invalidated it here for obvious reasons) and on the first glance you can see that this is not a PayPal domain. Shocker!

---

## Website

After that I clicked the link and visited the website (in a safe environment of course!) and was greeted with... a 404 page not found error. Well, great... So the phishing would not work even if someone fell for it.

I then navigated to the root of the website and found out that directory indexing was turned on:

![Index of /]({{site.baseurl}}/assets/2020/05/index_of_root.png)

As the link in the email was pointing to something in the Update directory, I navigated to it.

![Index of /Update]({{site.baseurl}}/assets/2020/05/index_of_update.png)

And here we can see a Login directory, which contains the actual phishing webpage — a login page that looks like a login page from PayPal. As with most phishing web pages you will come across, none of the buttons and links actually work except for the Next (or Login) button, which is another way to identify phishing.

![Phishing login page for PayPal]({{site.baseurl}}/assets/2020/05/Paypal-LogIn.png)

So we now know this is a phishing page and that it was poorly made, so we could end here and call it a day. But in the previous images you can see some files that could be interesting. So... let's have a look.

#### /Update/uni.php

First I started with the file `uni.php` in the Update directory and after visiting it, this message was shown:

![uni.php - first glance]({{site.baseurl}}/assets/2020/05/uni-php.png)

So this PHP script is used for unzipping a local archive file, which at first glance does not seem that interesting or useful. But nonetheless I tried to supply the script with a parameter `file` with the name of this script and this happened:

![uni.php - file parameter passed]({{site.baseurl}}/assets/2020/05/uni-php-file.png)

Alright, so we can seemingly unzip any file on the system. But what can we do with it? Probably nothing, let's move on to another file.

#### /mrben.php

So this file is another PHP script — as it turns out there was not much to it other than an input box for a password and a submit button.

![mrben.php - Login form]({{site.baseurl}}/assets/2020/05/mrben-php.png)

Well... I thought it would be interesting to look at those files but it was not... But there was another file, which by its name did not seem interesting but for the sake of completeness I took a look at it.

#### /404.php

As I expected this script from its name to be a "404 Not Found" page, I was surprised to be greeted with the following:

![404.php - Anonymous Shell Login form]({{site.baseurl}}/assets/2020/05/404-php.png)

Okay, this is getting interesting. But first let's recapitulate. The phishing author does not seem that sophisticated, so I assumed this would be a publicly available PHP web-shell. Probably on GitHub?

After hours and hours of searching through hundreds of pages... Okay, after a few seconds of Googling (the search term "anonymous shell github" worked) I found that the web-shell is available at [https://github.com/iamhex/Anonymous-Shell-v2](https://github.com/iamhex/Anonymous-Shell-v2). And if you read through the README, you can find:

> Password: hacker0882

Now you might be thinking: "No, this is not the real password. Surely the phishing author changed it!". Well as it turns out the opposite is true. The password was left at the default value and I was greeted with shell access to the server!

![404.php - Web Shell successfully accessed!]({{site.baseurl}}/assets/2020/05/404-php-shell-1024x301.png)

So now we have access to the server. We have entered a gray zone of law and there have been many discussions about what actions you can and cannot take. As I don't want to break the law (even in the slightest), I did not modify or delete anything — I only read the source code of the files we analyzed previously (which might still be a gray zone?).

---

## Source code

#### Phishing page

First I looked at the phishing page files, which were basic copies of the PayPal login page, along with a few PHP scripts I call "firewall" scripts — they block access from predefined hardcoded IP addresses and show a blank page instead.

The login page itself is divided into 4 steps:

1. The first step harvested login credentials (email and password) and the IP address of the victim.
2. The second step harvested payment card information and billing address of the victim.
3. The third step asked for an ID card from the victim.
4. The fourth step asked for a selfie of themselves.

The harvested information was saved to a file and emailed to the attacker's address at the end of each step. So even if the user stopped at step two, the attacker already had the login credentials.

#### /mrben.php

After that I looked at `mrben.php`. It turns out it is a PHP mailer — specifically TeamCC NinjaMailer. The login password was hardcoded in the script and this time it was changed from the default value.

#### /Update/uni.php

The last file I investigated was `uni.php`. As I found out, it unzips an archive on the local disk to the current directory, overwriting any existing files.

The interesting thing about this script is this line:

```php
system('unzip -o ' . $_GET['file']);
```

If you know a little PHP you may already see the problem. The author passes **anything** submitted in the `file` parameter directly to the PHP function `system()`, which executes it as an OS command. This introduces a **Command Injection** vulnerability.

So if we send a request with `uni.php; id` in the file parameter, the command `id` gets executed and its output is shown directly in the page:

![uni.php - Command Injection]({{site.baseurl}}/assets/2020/05/uni-php-id.png)

The commands run as the user under which the web server service is running, so privileges are limited to that user.

---

## Disclosure

After I found the phishing webpage I immediately reported the website to Google Safe Browsing ([https://safebrowsing.google.com/safebrowsing/report_phish/](https://safebrowsing.google.com/safebrowsing/report_phish/)). It did not take even a day to see it blocked in all major browsers.

![Google Safe Browsing stops a visit to the phishing website.]({{site.baseurl}}/assets/2020/05/Google-safe-browsing.png)

I also reported it to the owner of the IP address (found through the whois database). Only after that did I begin looking for more information, which this article came from. After finding more details through the web-shell I also reported it to the cloud provider, just in case.

---

## Conclusion

So this basic phishing email led us to discovering a publicly accessible web-shell with default credentials (see, even attackers forget to change them!) and we saw the whole process of gathering victim's credentials.

As you can see, phishing emails can sometimes be a source of fun — but only if you are careful!

Another great source of fun can sometimes be to reply to phishing and spam emails. As an example, I recommend reading this old but gold [article](http://news.bbc.co.uk/2/hi/africa/3887493.stm) about a scammer who got scammed by his victim — the story of the Church of The Order of The Red Breast. There are many more stories to be found on the internet if you look for them.

If you decide to try any of these I must warn you to always be very careful. I recommend replying from a dummy email address (not your main or secondary address) as attackers usually don't care if the reply comes from a different address than the one they originally targeted.
