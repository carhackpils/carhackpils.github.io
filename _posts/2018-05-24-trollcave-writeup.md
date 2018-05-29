---
layout: post
published: false
categories: write-up
title: Write-up Trollcave
excerpt_separator: <!--more-->
comments: true
---

In this post, I'll do the write-up of the [Trollcave](https://www.vulnhub.com/entry/trollcave-12,230/) VM. This is the first write-up I'm doing so sorry if I'm making some mistakes :p.

<!--more-->

For this write-up I will be using a Kali VM with VirtualBox. Kali and Trollcave are inside a NAT network created in VirtualBox called "LabNetwork"

First things first, let's find out where the IP of the Trollcave VM. For this I'm using `netdiscover` on the network 10.0.2.0/24.

{% highlight bash %}
root@kali:~# netdiscover -r 10.0.2.0/24

 Currently scanning: Finished!   |   Screen View: Unique Hosts                 
                                                                               
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240               
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.2.1        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.2.2        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.2.6        08:00:27:d0:d7:c5      1      60  PCS Systemtechnik GmbH 
{% endhighlight %}

One address has the VirtualBox MAC address prefix `08:00:27`. Next I'm doing a simple scan of this IP with `nmap`.

{% highlight bash %}
root@kali:~# nmap -A 10.0.2.6
Starting Nmap 7.70 ( https://nmap.org ) at 2018-05-24 16:59 EDT
Nmap scan report for 10.0.2.6
Host is up (0.00049s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:ab:d7:2e:58:74:aa:86:28:dd:98:77:2f:53:d9:73 (RSA)
|   256 57:5e:f4:77:b3:94:91:7e:9c:55:26:30:43:64:b1:72 (ECDSA)
|_  256 17:4d:7b:04:44:53:d1:51:d2:93:e9:50:e0:b2:20:4c (ED25519)
80/tcp open  http    nginx 1.10.3 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: nginx/1.10.3 (Ubuntu)
|_http-title: Trollcave
{% endhighlight %}

First, we can see that the SSH port is open, but I can't see any clue on how exploit it for now with the little information we have. However, there is a port `80` opened with a nginx web server running on it. Let's see whats going on there.

![trollcave_home.png]({{site.baseurl}}/images/trollcave/trollcave_home.png)

Yeay ! There is a website. In the nmap scan, we can see that there is also a robots.txt file. Robots.txt is a file that can be at the root of the website (let's not say server because every website can have its own robots.txt) and that tells to crawlers which paths are disallowed from being referenced from search engines.  Unfortunately for this website, it doesn't bring any new information to the case.

So, let's see now whats on that website. The first post we see is from a User named King. Under his username, it says that he is a super admin, it could be super nice if we could log in with that account. Since a user page has always the same format : http://<ip>/users/<id> we can script this to get every users. By deduction, there no users that have an ID more than 16 and these IDs are incremental. For your information, this is possible to do something similar with Burp Suite Community but I'd like to train myself to write more commands in the terminal.

{% highlight bash %}
root@kali:~# curl -s http://10.0.2.6/users/[1-20] | grep "'s page" | cut -d "'" -f 1 | sed -e "s/^<h1>//"
King
dave
dragon
coderguy
cooldude89
Sir
Q
teflon
TheDankMan
artemus
MrPotatoHead
Ian
kev
notanother
anybodyhome
onlyme
xer
{% endhighlight %}

Let's explain this command : 
- `curl -s http://10.0.2.6/users/[1-20]` : get all users page in silent mode (-s, no information except result) from ID 1 to ID 20.
- `grep "'s page"` : to get the line with the username
- `cut -d "'" -f 1` : get only the first part of the line with the username (cut on the quote character and take the first part)
- `sed -e "s/^<h1>//"` : sed is a stream editor and `-e` is to use it with a script. Here the script has this format `s/<1>/<2>/` and means that we `s`ubstitute <1> by <2>. So here we take the strings that begins with the `<h1>` tag and replace it by nothing.

Another interesting post is the one by **coderguy**. The post says :
> so far i've implemented a password_resets resource in rails and it's about 90% working except for the email thing.

This gives us two information :
1. Somewhere there is a buggy thing that can reset passwords (nice!)
2. This website is coded with Ruby on Rails (RoR)

I had absolutly no idea of how RoR works so I had to dig to find the reset password page (dirb was not really usefull for this one). Apparently, RoR have routes to connect paths on the website to a controller and an action. When you type the name of the ressource in the URL, a route will redirect you to the correct controller to execute the correct action. Here `password_resets` is a ressource if we believe **coderguy**, so I tried first to access the `index` action by only typing `http://10.0.2.6/password_resets`. No luck ! Then I tried the same URL but with the `new` action. This action, in most case, will display a form to create a new ressource. URL is then `http://10.0.2.6/password_resets/new`. Bingo ! Here is the form :

![trollcave_passwordreset.png]({{site.baseurl}}/images/trollcave/trollcave_passwordreset.png)

I said earlier that it could be super nice to have the King's user account. Let's try to reset his password. No luck again, we only can reset password for normal users. So let's try with a normal user. **xer** is the last member to register to the website and he is a normal user. I did the same process and this time a message tells me this :
> Reset email sent. http://10.0.2.6/password_resets/edit.kQAPr47NGOIBv4hUYQLdjA?name=xer

By accessing that URL I successfully reset xer password and connected to his account. That was too easy. But we are not super admin yet. With a closer look to the URL, we have a parameter called `name`. If we change this name, are we able to bypass the "normal user" restriction for password reset ? Obviously I'll try it with **King** username. The URL is : `http://10.0.2.6/password_resets/edit.kQAPr47NGOIBv4hUYQLdjA?name=King` (watch out, it's case sensitive) ... aaand here we go, I am **King** (and superadmin BTW).

![tumblr_msgganaCKF1sg0a6co1_500.gif]({{site.baseurl}}/images/trollcave/tumblr_msgganaCKF1sg0a6co1_500.gif)

What can I do from here ? Now that I have a full access on the website I could be nice that I move to the box directly. So this will be the next step. 
















