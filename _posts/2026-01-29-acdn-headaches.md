---
layout: post
title: "Apple CDN headaches"
date: 2026-01-30
---

I'm a big believer in Cunningham's Law:
> the best way to get the right answer on the internet is not to ask a question; it's to post the wrong answer.

Part of the reason I'm sharing some of my findings is in the hope that someone will appear with better solutions.


<br>
---


Recently I upgraded my internet subscription to a 1Gbps link.
Like any reasonable person would do, I tried to download an ipsw (iOS/Apple firmware image) with my higher bandwidth.

```
% curl -O https://updates.cdn-apple.com/2026WinterSeed/fullrestores/047-42574/0E1580E9-B95A-449E-B6E6-553FC521F908/iPhone17,3_26.3_23D5114d_Restore.ipsw
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0 10.0G    0 15.6M    0     0  2306k      0  1:16:21  0:00:06  1:16:15 2313k
```

??????
`2313k` (bytes per second)?

I stopped the download. Took a deep breath. Tried again - same issue.

I restarted my macbook (desperate people do desperate things); new download → still getting around 2MB/s.....

I picked a different update; perhaps that last beta wasn't cached yet; I'll pick something more recent → same issue.


After running more than 10 downloads I began seeing a few fast downloads interleaved between the pathetically slow ones:
```
 % curl -O https://updates.cdn-apple.com/2026WinterSeed/fullrestores/047-42574/0E1580E9-B95A-449E-B6E6-553FC521F908/iPhone17,3_26.3_23D5114d_Restore.ipsw
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  4 10.0G    4  495M    0     0  62.7M      0  0:02:44  0:00:07  0:02:37 59.7M
```

ok so occasionally the CDN is willing to serve me images at full speed but most of the time it's not and apparently I don't have any control over that.

2MB/s downloads of 10GB images give you a lot of time to think so I came up with a theory that **perhaps the Apple CDN server I'm connected to is broken/pathological**. Surely if I could download from another server, I could get reliably fast transfers.

<br>
---

Apple has got many customers all over the world that need to be able to download large update images.

It would be overwhelming to serve all of these updates from Cupertino only so they instead opt for a CDN with servers distributed around the world to reduce the sheer amount of bytes that have to be transferred (long distances).

On top of that, Apple partners with ISPs so that they (who are the closest to the end users) can host caching servers.
Apple's peering program is called [Apple Edge Cache](https://cache.edge.apple/); this is actually fairly common (Google, Netflix and the likes also do this).

The way Apple's CDN dispatches requests to its "edges" is through DNS; when a request comes in, the nameserver looks up the region (network?) of the source
and based on that it will reply with records for edge servers close to that source.

So when I try to resolve `updates.cdn-apple.com` from my home in Brussels:
```
% dig updates.cdn-apple.com +short
updates.g.aaplimg.com.
81.244.254.80
81.244.254.112
```

I receive IP addresses managed by my local ISP (Proximus) which is hosting some Apple Edge Cache servers. (note: `updates.cdn-apple.com` is technically an alias for `updates.g.aaplimg.com.`...)

If I use an IP from a Romanian ISP (using a VPN/proxy):
```
% dig updates.cdn-apple.com +short
updates.g.aaplimg.com.
82.78.25.243
```

**curl has got this very powerful option (`--resolve ...`) that lets you override the DNS lookup for a given host.**
(a similar outcome could also be achieved by adding an entry in `/etc/hosts`)
Using that I was able to download images from the Romanian edge servers (which were way more reliable than my home ISP's):

```
% curl \
--resolve updates.cdn-apple.com:443:82.78.25.243 \
-O https://updates.cdn-apple.com/2026WinterSeed/fullrestores/047-42574/0E1580E9-B95A-449E-B6E6-553FC521F908/iPhone17,3_26.3_23D5114d_Restore.ipsw
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  6 10.0G    6  708M    0     0  36.7M      0  0:04:41  0:00:19  0:04:22 51.1M
```

<br>
---

I remained puzzled by the occasionally fast downloads I was getting from my local edge cache servers.
One evening I began analyzing the HTTP headers that the servers were returning:

```
HTTP/1.1 200 OK
...
x-amz-meta-digest-sha256: 8edce65c6c2e0a59abcf10cc1d42d107f40c3ae1aba597c14569d2622cf69db3
...
Server: AmazonS3
X-Cache: Hit from cloudfront, hit-fresh, hit-fresh, none, hit-fresh, miss, none, miss, none
Via: 1.1 [TRUNCATED FOR BREVITY] http/1.1 nlams2-edge-bx-013.ts.apple.com (acdn/288.16361), http/1.1 proximus-bebru1a-fe-002.ec.edge.apple (acdn/260.16276)
...
```
mildly interesting finding: the CDN appears to run on CloudFront (using S3).
we receive some info about the cache hits/misses along the chain of CDN servers and the `Via` header lists them out.

aha moment: I spotted a pattern: all of the garbage slow downloads were routed through `proximus-bebru2a-fe-002.ec.edge.apple`
If they were routed through `proximus-bebru1a-fe-002.ec.edge.apple` instead, they were completely fine!!!

I resolved the IP address of the `bebru1a` server, used that as my target address for curl's `--resolve` and voilà: no more unreliable downloads (at least so far).

Unfortunately I don't know enough about how Apple's Edge Caches and my ISP's network operate to have a clue as to why one of those edges seems to cap the download speed so aggressively.


<br>
---

I wanted to find more (reliable) edge servers but the way they're selected through DNS makes it relatively difficult (essentially requiring an IP in a different part of the internet for each request).

Ideally, I'd want to be able to find all of the DNS A records that map to `updates.g.aaplimg.com.`.

There are some services out there that gather data (including DNS records) from internet scanners. I managed to find some IPs by searching for `ec.edge.apple` on Shodan.
It's worth noting that the vast majority of those edge servers are peers managed by ISPs.

Coincidentally I found some CDN servers on Apple IPs by looking at `networkQuality(1)`'s endpoints, obtained through mensura.cdn-apple.com

```
% dig mensura.cdn-apple.com +short
mensura.lb-apple.com.akadns.net.
mensura.g.aaplimg.com.
17.253.53.206
17.253.53.201
```

<br>
---

With a couple dozen CDN server IPs, I wanted to see if I could maximize my download speed. I turned to [aria2](https://aria2.github.io/).

I had Claude throw together a shell script that uses `aria2c` to download an image from different mirrors in parallel.

aria2 can also gather statistics on different servers to select for the fastest mirrors (in subsequent downloads).
Unfortunately aria2 doesn't have an option similar to curl's `--resolve`, as such TLS checks would fail (SNI mismatch); I get around this by fetching the checksum using curl and checking that it matches once the download was completed.

Here's what the invocation of aria2c looks like in the script:
```
aria2c \
  --check-certificate=false \
  --file-allocation=falloc \
  --header='Host: updates.cdn-apple.com' \
  --split=12 \
  --max-connection-per-server=12 \
  --uri-selector=adaptive \
  --server-stat-if="$STAT_FILE" \
  --server-stat-of="$STAT_FILE" \
  --out="$FILENAME" \
  $URIS
```
And here it is chipping away (I'm using a 1Gbps ethernet to usb-c adapter so these speeds are not mindblowing either).
```
[#83fb8c 1.6GiB/10GiB(15%) CN:12 DL:111MiB ETA:1m17s]
```

I posted the script in a [gist](https://gist.github.com/hde-moff/65eb1699e1fd29b92af86435d9d36d0c#file-aria2c_dl-sh).
