My server is running Slackware 14.1 x64.

Linux mx10 3.10.40 #2 SMP Tue May 27 13:28:40 BST 2014 x86_64 Intel(R) Xeon(R) CPU           X3220  @ 2.40GHz GenuineIntel GNU/Linux

I added HAVE_IPV6=YES to the Local/Makefile as per Chapter 4.9 of the documentation.  The exim binary builds without any errors and executes, but when I telnet on port 25 to mx.mydomain.co.uk it fails to respond with the banner message, even on IP4.

I had to recompile without IPv6 to restore service.

220 mydomain.co.uk ESMTP Exim 4.82 Wed, 02 Jul 2014 12:07:22 +0100

It mentions in Chapter 4.9 "it may also be necessary to set IPV6_INCLUDE and IPV6_LIBS".  Where would I find these?  What should I be looking for?  Other applications (e.g. ntpd, httpd, named) are all executing fine on IPv6 and I can connect to other IPv6 hosts OK (e.g. www.kame.net dancing turtle).