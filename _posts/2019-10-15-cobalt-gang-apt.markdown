---
layout: post
title:  "10/15/2019 - Cobalt Gang APT: Recent infrastructure and CobInt/COOLPANTS malware analysis"
date:   2019-10-15 00:00:00 -0700
categories: tech
author: MELTX0R
---
<center><img src="{{site.baseurl}}/assets/images/cobaltGangBanner.jpg" style="max-width:100%;max-height:100%;"></center>

&nbsp;

## Summary

Cobalt Gang (also known as Cobalt Group or Cobalt Spider) is a financially motivated threat group that has largely targeted financial institutions. According to [MITRE](https://attack.mitre.org/groups/G0080/) and other security organizations, the group has primarily targeted banks in Eastern Europe, Central Asia, and Southeast Asia. The activity discussed in this analysis relates to the CobInt/COOLPANTS malware, which has been attributed as a tool utilized by Cobalt Gang.

&nbsp;

## Analysis


While performing research, I came across an interesting document titled *"PFD-19-010.doc"* being hosted on the URL *www.relax-cream.com/wp-content/plugins/Boss/PFD-19-010.doc*. This document purported to be from the Visa, and contained material meant to invoke a concern-driven action by the recipient, such as "Payment Fraud Disruption". The document prompted the recipient to enable Editing/Content to view the "protected" document.

&nbsp;

<center><img src="{{site.baseurl}}/assets/images/COBALTGANG_VISA_DOC_LURE_10152019.png" style="max-width:100%;max-height:100%;"></center>
<span style="font-size:small;"> Shown above: Visa themed lure used by Cobalt Gang</span>

&nbsp;

By enabling Editing/Content, the password-protected macro is able to run - this will drop the file *"error_log.vbe"* in the user's local temp directory and execute the script via *WScript.exe*. While I wasn't able to decode the script itself, upon execution it would manipulate Windows Certificates by writing a blob to *HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\SystemCertificates\AuthRoot\Certificates\\*. Using the *CertUtil* utility I was able to decode this blob to readable text, which presented the below information.

&nbsp;

{% highlight text %}

================ Certificate 0 ================
================ Begin Nesting Level 1 ================
Serial Number: 44afb080d6a327ba893039862ef8406b
Issuer: CN=DST Root CA X3, O=Digital Signature Trust Co.
 NotBefore: 9/30/2000 5:12 PM
 NotAfter: 9/30/2021 10:01 AM
Subject: CN=DST Root CA X3, O=Digital Signature Trust Co.
Signature matches Public Key
Root Certificate: Subject matches Issuer
Template:
Cert Hash(sha1): da c9 02 4f 54 d8 f6 df 94 93 5f b1 73 26 38 ca 6a d7 7c 13
----------------  End Nesting Level 1  ----------------
No key provider information
Cannot find the certificate and private key for decryption.
{% endhighlight %}
&nbsp;


I am unsure of the significance of this certificate at this time, as it is not utilized by any infrastructure I've identified up to this point. Following this, the script would download a payload from the URL *"www.huanchacosurf.inti.co.uk/vendor/bin/avatar.hlpv"*, store it in the user's local temp directory, rename it as *"Colors.exe"*, and execute it. Interestingly this binary was compiled on *October 13th 2019 at 17:14:27* and purports to be signed by *Symantec Corporation*, however it fails verification.

&nbsp;


<center><img src="{{site.baseurl}}/assets/images/COBALTGANG_PAYLOAD_CERT_10152019.png" style="max-width:100%;max-height:100%;"></center>
<span style="font-size:small;"> Shown above: Certificate information of payload purporting to be signed by Symantec</span>

&nbsp;


Following execution of *"Colors.exe"*, Command & Control would then be initiated to *hunvenbinusa.info* over TCP/443. After initial C2 is established, the data returned is stored in a text file titled *"zvdpoaqrvytayoaygk[1].txt"* in the following location:
*C:\Users\[USERNAME]\AppData\Local\Microsoft\Windows\Temporary Internet Files\Content.IE5\LH043OAM\zvdpoaqrvytayoaygk[1].txt*. This file, while at first appearing to be a benign HTML file based on various HTML tags, in fact contains commands sent by the Command & Control server.

&nbsp;


<center><img src="{{site.baseurl}}/assets/images/COBALTGANG_CMD_TXT_FILE_10152019.png" style="max-width:100%;max-height:100%;"></center>
<span style="font-size:small;"> Shown above: Contents of zvdpoaqrvytayoaygk[1].txt</span>

&nbsp;


At this point in my investigation, I was confident that the payload I was analyzing was CobInt/COOLPANTS (or similar variant) malware used by Cobalt Gang. According to an article released by [ProofPoint](https://www.proofpoint.com/us/threat-insight/post/new-modular-downloaders-fingerprint-systems-part-3-cobint) in September 2018, the text file contains encrypted command data - such as commands to load/execute a module, stop polling the C2, execute a function set by a module, or update the C2 polling wait time. To decrypt this data, ProofPoint states that you would need to:


1. Remove HTML tags
2. Convert all text to lowercase
3. Remove all characters that are not “a-z”
4. Convert the characters into binary data via an unknown decoding algorithm
5. XOR decrypt the binary data with the embedded 64-byte XOR key used in C&C host decryption
6. Perform a second round of XOR decryption using the following key:
 * XOR key length is indicated by the last byte of data
 * XOR key is the last “X” bytes of data (excluding length byte), where “X” is the length of the key

  &nbsp;



They also provided a Python script to automate this process, which can be found [here](https://github.com/EmergingThreats/threatresearch/blob/master/cobint/stage2_decrypt_response.py) on GitHub.

&nbsp;


I was unable to identify any subsequent commands or modules being loaded throughout my analysis of the payload. With that in mind, I decided to try and identify additional infrastructure being utilized by Cobalt Gang for this campaign, which is where I started to go down a rabbit hole...

&nbsp;


<center><img src="{{site.baseurl}}/assets/images/COBALTGANG_VT_GRAPH_10152019.png" style="max-width:100%;max-height:100%;"></center>
<span style="font-size:small;"> Shown above: VirusTotal Graph of investigation</span>



&nbsp;


Performing a search on PassiveTotal for the first domain I observed *(www.huanchacosurf.inti.co.uk)* revealed 34 additional subdomains. Initial research into these subdomains made them appear as if they were business pages in various fields such as law, media, outdoors, and psychology - all of which were located in *Trujillo Peru*. An additional point of interest is that *"inti.co.uk"* is very similar to the legitimate *"init.co.uk"* domain, which belongs to the German based company *"INIT Group"*.

&nbsp;

<span style="font-size:small;"> Shown below: Subdomains for INTI.CO.UK</span>
{% highlight text %}

ascoyabogados.inti.co.uk
barriosanjose.inti.co.uk
brallec.inti.co.uk
ceramicoshuanchaco.inti.co.uk
easyclubadmin-net.inti.co.uk
ftp.inti.co.uk
huanchacosurf.inti.co.uk
inti.co.uk
ladrilloschanchan.inti.co.uk
mail.inti.co.uk
me.inti.co.uk
moromeinmobiliaria.inti.co.uk
nirvan.inti.co.uk
nirvana.inti.co.uk
psicoaccion.inti.co.uk
renacerfuneraria.inti.co.uk
sbssanjorge.inti.co.uk
screenmediastudio.inti.co.uk
sermedicsac.inti.co.uk
surfcastingtrujillo.inti.co.uk
www.ascoyabogados.inti.co.uk
www.barriosanjose.inti.co.uk
www.brallec.inti.co.uk
www.ceramicoshuanchaco.inti.co.uk
www.easyclubadmin-net.inti.co.uk
www.huanchacosurf.inti.co.uk
www.ladrilloschanchan.inti.co.uk
www.me.inti.co.uk
www.moromeinmobiliaria.inti.co.uk
www.psicoaccion.inti.co.uk
www.renacerfuneraria.inti.co.uk
www.sbssanjorge.inti.co.uk
www.screenmediastudio.inti.co.uk
www.sermedicsac.inti.co.uk
www.surfcastingtrujillo.inti.co.uk



{% endhighlight %}


&nbsp;


Reverse DNS searches into the IP address hosting *inti.co.uk (173.254.28.36)* revealed that there were actually *".com"* versions of these domains as well - such as *surfcastingtrujillo.com* or *sermedicsac.com*. Reviewing these domains returned some interesting findings - such as some of them having the same favicon, lorem ipsum text, fake reviews, the same addresses, and more. These domains also contained text indicating they were developed by a Peruvian media company "Screen Media Studio". Research into this media company returns a YouTube channel, a website (*screenmediastudio.com*), Facebook page, and more, and appeared to be legitimate as a result.

&nbsp;


At this point in my investigation, I was starting to think that these domains were unrelated to the infrastructure I was researching. I then performed a final search on PassiveTotal to see what SSL certificates *inti.co.uk* used and found four *"LetsEncrypt"* certificates that were recently used by all the aforementioned domains *INCLUDING* *screenmediastudio.com*. At this point in time, I have not seen any subdomain besides *"www.huanchacosurf.inti.co.uk"* serve malicious artifacts for this campaign, and therefore cannot confirm that the other listed domains are related to or being used by this campaign, but I find the aforementioned similarities highly anomalous, including that all of the *inti.co.uk* subdomains and seemingly "legitimate" domains share the same LetsEncrypt certificates as a domain serving CobInt/COOLPANTS malware.

&nbsp;



|Serial Number | Issued | Expires |
|220b91fa140101dde6fe1d9102fb19c922458a42 | 2019-09-27 | 2019-12-26 |
|b6e1290d270c0bd0573f73d8c022efc176fa9d4a | 2019-09-27 | 2019-12-26 |
|47062ed4b342879f5e6a53cd3826be942a8f0f1d | 2019-09-01 | 2019-11-30 |
|83cd57a38ca395623a4d7481e0305f8f6b645aee | 2019-09-01 | 2019-11-30 |

<span style="font-size:small;"> Shown above: LetsEncrypt Certificates used by inti.co.uk recently</span>

&nbsp;



I then performed a search on PassiveTotal for the initial C2 domain *(hunvenbinusa.info)* and found that it shared an IP address with two other domains - *bueatyslim.site* and *ispot-world.com*. While I haven't found evidence indicating *ispot-world.com* is related to this campaign, *Censys.io* records reveal that *bueatyslim.site* utilizes a LetsEncrypt certificate that contains a SANs (Subject Alternative Name) of *"hunvenbinusa.info"* - therefore, I believe *bueatyslim.site* to be an additional C2 domain utilized in this campaign.


&nbsp;


At this time, I was unable to obtain evidence of target attribution - however they have primarily targeted financial institutions in Eastern Europe, Central Asia, and Southeast Asia per [MITRE's](https://attack.mitre.org/groups/G0080/) research. It is also interesting to see how investigating small "threads" of evidence can lead to going down many rabbit holes - possibly unraveling the "ball of yarn" that is an attacker's infrastructure.


&nbsp;

## Indicators

|Indicator|	Type	|Description|
| www.relax-cream.com/wp-content/plugins/Boss/PFD-19-010.doc | URL | URL serving PFD-19-010.doc |
| 8e8e7b25a0df0dfed26d726cb1c01567 | MD5| PFD-19-010.doc - Visa themed .Doc lure containing embedded macro leading to download of CobInt/COOLPANTS malware |
| www.huanchacosurf.inti.co.uk/vendor/bin/avatar.hlpv | URL | URL serving CobInt/COOLPANTS malware |
| 6ef835a8ac1cc70d4b478c7c45efa5db| MD5 | Colors.exe - CobInt/COOLPANTS malware hash |
| hunvenbinusa.info | Domain | CobInt/COOLPANTS Command & Control server |
| bueatyslim.site | Domain | CobInt/COOLPANTS Command & Control server |




&nbsp;

## References/Further Reading

1. https://www.proofpoint.com/us/threat-insight/post/new-modular-downloaders-fingerprint-systems-part-3-cobint
2. https://github.com/EmergingThreats/threatresearch/blob/master/cobint/stage2_decrypt_response.py
3. https://threatpost.com/cobalt-group-targets-banks-in-eastern-europe-with-double-threat-tactic/137075/
4. https://attack.mitre.org/groups/G0080/
5. https://app.any.run/tasks/f6e0598d-a1e4-4503-953c-6d7506e2ef66/
