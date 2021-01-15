---
layout: post
title: Outlook 2016 hanging on splash screen
date: 2018-05-31 14:24:46.000000000 +02:00
type: post
published: true
categories:
- Office
- Outlook
- Wireshark
tags:
- issue
- migrated
- proxy
- slow
permalink: "/2018/05/31/outlook-2016-hanging-on-splash-screen/"
---

In the last days I was involved in finding the reason for an odd issue with a hanging / slow starting Outlook 2016\. Users reported that Outlook 2016 is starting really slow after restarting the computer. Closing and opening Outlook again is working as expected.

![outlook2016_splash]({{ site.baseurl }}/assets/migrated/2018/05/outlook2016_splash.png)

The issue does not occur always after restarting the computer, so pinning it down was really nasty. We had some clients where we could reproduce the behavior quite well, but on other clients it was pure luck to see this happen when we connected to the client machine.

Colleagues of mine started to capture network traffic with Wireshark from the affected computers and send it to me. In these capture files I was inspecting every communication between the client and the remote endpoints. What I could see was, there are time spans in wich Outlook is not sending any tcp packet (this lastet at least  for 30 seconds) - So I could confirm the reports of a non responding Outlook just by looking to these dumps! But there was no clue on what exactly was preventing the application from communicating with the backend.

Some of my first checks and the results:

*   **DNS resolving** - working
*   **Proxy exception** - in place; communication went directly to the load balancer
*   **TCP Resets** - none found

While I was still looking into the captures my teammates told me they discovered something. They found a workaround that was working quite well. We are using a pac file for proxy configuration that is configured by GPOs in our environment. Auto detecting proxy settings is turned off.  The observation from my colleagues was, after restarting the client and changing the proxy from using a pac file to manual proxy configuration solves the problem and Outlook 2016 starts quickly as supposed.

Everyone (including me) was thinking: This has to be some what related to the proxy pac file..... But we all where sooo wrong :-)

While my teammates where looking into this new discovery I digged through the capture files again and was falling almost asleep while my eyes where following row after row of packets. There is nothing! -I closed Wireshark and tried a short google query instead - there must be someone who has struggled with the same issue before. My first search brought me directly to the blog of Kevin Street:

[https://kevinstreet.co.uk/2018/01/30/outlook-2016-slow-loading-issue/](https://kevinstreet.co.uk/2018/01/30/outlook-2016-slow-loading-issue/)

Immediately I was thinking: **"Man, this is something! - Maybe we also have a long list of DNS suffixes in this domain and struggling with WPAD?"**

Next day I talked to my Active Directory team and we looked in to the GPOs and also in the DNS of this particular domain. First we discovered a slightly misconfigured DNS suffix list - but this wasn't a big deal. We found only two entries in this list - so way easier to analyze than the situation of Kevin.

Looking into the DNS we found an entry for WDAP. Wow! - Immediately I have asked: "What host or IP it is pointing to?". The moment they told me the IP address, I laughed out loud. I knew this IP - I saw it a lot of times in my nightly Wireshark sessions - but I did not put it in the right context to WDAP! Why this?

![wiresharkdump.png]({{ site.baseurl }}/assets/migrated/2018/05/wiresharkdump.png)

1.  I have seen a tcp transmission for SYN packet going to the this IP, but never a SYN,ACK - This was strange! So I have looked into our CMDB database for this particular IP and found an asset with the service description "Network monitor server". Ok - a server that should monitor something; maybe there is some software installed on the client that is trying to send some health information? There was also a note in the CMDB, that there is installed a printing management software, too - Could also be possible that there is something related with printing. But nothing relates to WPAD, DNS, Exchange or Proxies. So my brain created a cognitive bias for this IP :-)
2.  Every time I got a new dump, one of my first filter queries was 'dns'. And in all of the dumps there was no client side lookup for the WPAD entry. This was mainly because Wireshark was started too late after client restart and the lookup to WPAD was already done before.

After this discovery we removed the misleading WPAD entry from DNS and next day the issue was resolved and Outlook 2016 started perfectly normal.

Afterwards we found out that the Server with the wpad.dat file on gone lost in a migration some weeks ago. The next puzzle part that lead to the issue was the roll out of Outlook 2016 two Weeks ago. So Outlook 2016 defiantly has a different behavior in comparison to Outlook 2010 in regards to WPAD. Maybe Microsoft took into consideration the high demand to a functional internet connection nowadays. So waiting for the proxy configuration might be better than having to show error a dialog that Exchange Online is not available?!?

At the end there are some conclusion for me I learned from this session (again :-) )

1.  A good maintained documentation/CMDB is pricless
2.  Always flush your dns cache before tracing with Wireshark
3.  Ask google the right questions
4.  Share your knowledge with others (Thank you again Kevin for doing exactly that!)

I am still asking myself, why did setting the proxy from a script to manual solved the problem as a workaround. My explanation is, that after restarting the client the proxy pac file configuration is applied by the GPO. Before starting Outlook the user has manually changed the proxy settings. Maybe changing this setting takes so much time that the WPAD lookup in the background already gave up to contact the endpoint. Or switching the proxy settings interrupts the lookup immediately and starting Outlook works as expected.