# APT exploration: SSL Masquerading

## Background

Back in 2009 Google and several other large US-based corporations suffered a highly sophisticated cyber attack dubbed "Project Aurora". According to McAfee, the attack was conducted by a large group of cyber criminals dubbed "[APT 17](https://attack.mitre.org/versions/v16/groups/G0066/)" (Alternatively knows as "Elderwood", "Axiom" or "The Beijing Group". The group is suspected to be a Chinese cyber espionage group that frequently executes highly sophisticated cyber attacks using a variety of techniques. Project Aurora was notably executed using a zero-day vulnerability in Internet Explorer which allowed a single link to download and execute malware. While the IE zero day is sufficiently interesting I'm more interested in one of the persistence techniques they used. The malware installed via the zero day was called [Hydraq](https://attack.mitre.org/versions/v16/software/S0203/) and performs a variety of different operations. One of which is "masquerading as the SSL/TLS protocol", which I'll be exploring in more detail further. The primary motivation for this is that once a RAT like Hydraq is on the system, the system is should be assumed to be fully compromised, meaning threat detection coming locally from said system can not be trusted. This leaves network based Intrusion Detection Systems (IDS) as the most reliable way for detecting a system compromised by something like this but by masquerading as SSL/TLS communication Hydraq makes that easier said then done. 

## What is the SSL Protocol?
-SSL or Secure Socket Layer is older version of the TLS (Transport Layer Security) protocol. Basically its a short back and forth communication between two computers that establishes what encryption algorithms will be used and that the sender and recipient are who they say they are. The primary difference between TLS versions are what algorithms are accepted by the server and client. 

## Why is this interesting?
Personally, I find this interesting because TLS/SSL traffic is almost always expected to be encrypted and therefor has an easier time evading intrusion detection systems. If you're scanning network traffic, and you're expecting gibberish on a specific port, and you're getting gibberish on said port, its near impossible to determine if the traffic is malicious or not.  Because of this, I wanted to understand in more detail what Hydraq actually does regarding its communication with C2 and how that ultimately could be detected. And what better way to understand a technique then to isolate and implement it yourself? 

## Chatting with Hydraq
Thanks to the wonderful work done by Sergei Shevchenko in his [blog](http://blog.threatexpert.com/2010/01/trojanhydraq-part-ii.html). Hydraq C2 communications have been clarified. First it attempts to resolve an obfuscated host name, which if that fails queries a DNS server for an alternative (legitimate) host name. If this fails, it goes to sleep and tries again in 2 minutes. If it can resolve the hostname it sends a 20 byte packet to the remote host on port 443 (the port ssl/tls traffic usually communicates on). The package is the following:
```
00 00 00 00 00 00 FF FF 01 00 00 00 00 00 00 00 00 00 77 00
```

Which is encoded via the "NOT" operation to 
```
FF FF FF FF FF FF 00 00 FE FF FF FF FF FF FF FF FF FF 88 FF
```

When the packet is submitted to the C2 server, its response is a 20 byte long packet that contains a number between 0 and 18, inclusive. this number is then "encrypted with the XOR 0xCC". The last line of Sergei's description is a little unclear but I'll assume it either means each byte is XOR'd with 0xCC or the full packet is XOR'd with 0xCC.  I'm assuming the former for implementation purposes however since a single XOR encryption is incredibly primitive, any further analysis should work either interpretation. This would mean a C2 package would look like:

``` 
# return message '0'
CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC
```
to
```
# return message '18'
CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC DE
```
This is actually rather peculiar, since this kind of encryption is roughly just as secure as a Caesar Cipher the goal is likely temporary obfuscation over anything else. However by the very nature of such a repeatable pattern, it is distinctly different from random values. This distinction can lead to straight forward, network based detection based on calculated entropy (a value that differs greatly depending on how "predictable" the sequence is). Since the majority SSL/TLS traffic is about as random as computers can be this will stick out like a sore thumb and developing automatic detection for C2 traffic to the Hydraq RAT is almost trivial as a consequence. 
## Experiment
A simple experiment designed to isolate this kind of traffic and compare what the network traffic would look like compared to usual SSL traffic can be conducted in 3 parts:
- Visit a few legitimate HTTPS webpages 
- Build a basic client and server that bind to port 443 and mimics the Hydraq RAT to C2 communications
- Capture packages from both experiments using wireshark and compare the observed results. 

The purpose of this experiment is to try to understand the caveats of packet based detection methods, a notable data point would be non-random traffic on port 443 from legitimate sources which might throw a wrench in entropy based detection methods. 
## Network based detection methods

## Closing remarks 

https://en.wikipedia.org/wiki/Operation_Aurora
https://attack.mitre.org/software/S0203/
http://blog.threatexpert.com/2010/01/trojanhydraq-part-ii.html
https://en.wikipedia.org/wiki/Transport_Layer_Security
https://www.ssldragon.com/blog/https-port-443/
