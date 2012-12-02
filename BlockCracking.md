## Blocking compromised accounts and auth cracking
Nowadays users' passwords often are stolen (with drive-by exploits, Windows malware, phishing) and used for spamming. Spam sent with authentication via your server causes it to be blacklisted without notice and sometimes no appeal. Simple rate limiting authenticated users constrains honest users while still allowing spam to trickle through, your server still ends up in blacklists. Each server needs automatic detection and blocking of compromised accounts (stolen passwords). I amended and implemented (for Exim version 4.67 or higher) Andrew Hearn's [idea](https://lists.exim.org/lurker/message/20100226.153132.58ab2e98.en.html) to check not rate of messages or all recipients, but rate of attempts to send to nonexistent recipient email addresses. Vast majority of spammers never try to validate every recipient address. Spammers harvest strings looking like email addresses from webpages and disks of trojaned Windowses, then sell huge lists of email addresses to each other. These lists contain very much email addresses which don't exist anymore or never existed: Message-Ids, corrupted strings in memory and files. In short, spammers' lists of email addresses are much dirtier than lists honest users send to. Honest users are very unlikely to attempt to send to 100 nonexistent email addresses in one hour. Below I explain in detail (for novices at Exim) what to change in
Exim config for automatic blocking of compromised and spammers' accounts, with automatic email notification to sysadmin or your abuse or support staff.

This code also blocks brute force password cracking via SMTP (it's not as important but a little useful).

Replace the paragraph with the line `accept  authenticated = *` with three paragraphs:

      accept authenticated = *
            set acl_m_user = $authenticated_id
            condition = ${if exists{$spool_directory/blocked_authenticated_users}}
            condition = ${lookup{$acl_m_user}lsearch\
                        {$spool_directory/blocked_authenticated_users}{1}{0}}
            control = freeze/no_tell
            control = submission/domain=
            add_header = X-Authenticated-As: $acl_m_user
    
      accept authenticated = *
            !verify = recipient/defer_ok/callout=10s,defer_ok,use_sender
            ratelimit = LIM / PERIOD / per_rcpt / user-$acl_m_user
            continue = ${run{SHELL -c "echo $acl_m_user \
               >>$spool_directory/blocked_authenticated_users; \
               \N{\N echo Subject: user $acl_m_user blocked; echo; echo because \
               has sent mail to LIM invalid recipients during PERIOD.; \
               \N}\N | EXIMBINARY WARNTO"}}
            control = freeze/no_tell
            control = submission/domain=
            add_header = X-Authenticated-As: $acl_m_user
    
      accept authenticated = *
            control = submission/domain=`

If the line `hostlist   relay_from_hosts =`
contains something besides `localhost` or `127.0.0.1`
or this server's IP-address, i.e. your server is a relay for
your LAN or users on your company's IP-addresses,
then replace the paragraph with the line
`accept  hosts         = +relay_from_hosts`
with three paragraphs:

      accept hosts = !@[] : +relay_from_hosts
            set acl_m_user = $sender_host_address
                             # or username from RADIUS
            condition = ${if exists{$spool_directory/blocked_relay_users}}
            condition = ${lookup{$acl_m_user}lsearch\
                        {$spool_directory/blocked_relay_users}{1}{0}}
            control = freeze/no_tell
            control = submission/domain=
            add_header = X-Relayed-From: $acl_m_user
    
      accept hosts = !@[] : +relay_from_hosts
            !verify = recipient/defer_ok/callout=10s,defer_ok,use_sender
            ratelimit = LIM / PERIOD / per_rcpt / relayuser-$acl_m_user
            continue = ${run{SHELL -c "echo $acl_m_user \
               >>$spool_directory/blocked_relay_users; \
               \N{\N echo Subject: relay user $acl_m_user blocked; echo; echo \
               because has sent mail to LIM invalid recipients during PERIOD.; \
               \N}\N | EXIMBINARY WARNTO"}}
            control = freeze/no_tell
            control = submission/domain=
            add_header = X-Relayed-From: $acl_m_user
    
      accept hosts = +relay_from_hosts
            control = submission/domain=

Insert into beginning of config:

    acl_smtp_connect = acl_check_connect
    acl_smtp_auth = acl_check_auth
    acl_smtp_mail = acl_check_mail
    acl_smtp_quit = acl_check_quit
    acl_smtp_notquit = acl_check_notquit
    LIM = 100
    PERIOD = 1h
    WARNTO = abuse@example.com
    EXIMBINARY = /usr/local/sbin/exim -f root
    SHELL = /bin/sh

In the WARNTO line replace `abuse@example.com` with your
abuse or support or sysadmin email address;  
I specified paths in EXIMBINARY and SHELL lines for FreeBSD,
adjust for your operating system.

Immediately after the "begin acl" line insert:

    acl_check_auth:
      drop  message = authentication is allowed only once per message in order \
                      to slow down bruteforce cracking
            set acl_m_auth = ${eval10:0$acl_m_auth+1}
            condition = ${if >{$acl_m_auth}{2}}
            delay = 22s
    
      drop  message = blacklisted for bruteforce cracking attempt
            set acl_c_authnomail = ${eval10:0$acl_c_authnomail+1}
            condition = ${if >{$acl_c_authnomail}{4}}
            continue = ${run{SHELL -c "echo $sender_host_address \
               >>$spool_directory/blocked_IPs; \
               \N{\N echo Subject: $sender_host_address blocked; echo; echo \
               for bruteforce auth cracking attempt.; \
               \N}\N | EXIMBINARY WARNTO"}}
    
      accept
    
    acl_check_quit:
      warn  condition = ${if def:authentication_failed}
            condition = $authentication_failed
            logwrite = :reject: quit after authentication failed: \
                                ${sg{$sender_rcvhost}{\N[\n\t]+\N}{\040}}
            ratelimit = 7 / 5m / strict / per_conn
            continue = ${run{SHELL -c "echo $sender_host_address \
               >>$spool_directory/blocked_IPs; \
               \N{\N echo Subject: $sender_host_address blocked; echo; echo \
               for bruteforce auth cracking attempt.; \
               \N}\N | EXIMBINARY WARNTO"}}
    
    acl_check_notquit:
      warn  condition = ${if def:authentication_failed}
            condition = $authentication_failed
            logwrite = :reject: $smtp_notquit_reason after authentication failed: \
                                ${sg{$sender_rcvhost}{\N[\n\t]+\N}{\040}}
            condition = ${if eq{$smtp_notquit_reason}{connection-lost}}
            ratelimit = 7 / 5m / strict / per_conn
            continue = ${run{SHELL -c "echo $sender_host_address \
               >>$spool_directory/blocked_IPs; \
               \N{\N echo Subject: $sender_host_address blocked; echo; echo \
               for bruteforce auth cracking attempt.; \
               \N}\N | EXIMBINARY WARNTO"}}
    
    acl_check_mail:
      accept set acl_c_authnomail = 0
    
    acl_check_connect:
      drop  message = $sender_host_address locally blacklisted for a bruteforce \
                      auth (username+password) cracking attempt
            condition = ${if exists{$spool_directory/blocked_IPs}}
            condition = ${lookup{$sender_host_address}lsearch\
                        {$spool_directory/blocked_IPs}{1}{0}}
    
      accept

When your staff receives a message with Subject like
`115.150.81.95 blocked`, just check that the IP-address is
unknown for you (in this example China) - it's who attempted
to crack passwords.

When your staff receives a message with Subject like
`user johndoe@yourisp.com blocked`, it means that using this user's
password multiple messages were sent through your server,
and during last hour 100 of recipient email addresses were rejected 
(5xx) by recipient MXs. This is very unlikely with honest users,
but typical for spammers. You can look in your logs
which recipient email addresses were rejected by your Exim:

    cd /var/log/exim;
    zcat mainlog*.* | fgrep 'Unknown user' | fgrep -v 'sender verify fail' | sed -E -e 's/^.+<.*<//' -e 's/>.+$//' | sort | uniq -c | sort -nr | less

Besides role accounts nonexistent on your domains like
webmaster@, sales@, office@, you'll see Message-IDs,
pieces of email addresses, pieces glued with random pieces of words.
For example, there were 540 attempts to spam a never existing address
on my domain `x-originating-ipa@lena.kiev.ua`.
Spammers will try to send through your relay (using passwords
stolen from your users) to multiple nonexistent addresses too.
When your staff receives notification with `Subject: user ... blocked`,
using `exipick` command (it's installed with Exim) look at content of messages
frozen in the queue sent with the username specified in the notification,
for example:

    exipick -zi '$h_X-Authenticated-As eq johndoe@yourisp.com' | xargs -n 1 exim -Mvc | less

If you see not spam (very unlikely) then using a text editor
delete the line with that username from the file `blocked_authenticated_users`
in Exim spool directory, for example `/var/spool/exim/` in FreeBSD
(if the file contains only one line which is likely
then you can just delete the file instead)
and unfreeze detained messages:

    exipick -zi '$h_X-Authenticated-As eq johndoe@yourisp.com' | xargs exim -Mt

If you see spam then change the user's password, notify the user
and delete the line or file (see previous paragraph).
In order for the user to really get it,
provide a clause in the user agreement or contract beforehand:
if the user's password was used for spam (no matter who spammed -
the user or somebody else) then the user pays a fine.
Spam frozen in the queue is evidence.
When the evidence is not needed anymore, delete the frozen spam:

    exipick -zi '$h_X-Authenticated-As eq johndoe@yourisp.com' | xargs exim -Mrm

Similarly with users on LAN or the ISP's IP-ranges
(file `blocked_relay_users`,
notifications with Subject like `relay user 192.168.12.34 blocked`).

[Lena](Lena)