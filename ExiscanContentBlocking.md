Author: Silmar A. Marca

1) To block spyware and dangerous content, use in acl\_check\_mime. 
Executable files: (html or text format: http[s]://\<virus
site\>/executable.exe)

Before, set these in Main Configuration Settings:

EXT_Executaveis_Link	= exe|bat|cmd|ws|wsc|wsf|wsh|js|jse|vb|vbe|vbs|pif|lnk|url|msi|hta|scr|cpl|reg|ins|isp| \
			  cab|dll|vxd|ole|ocx|sys|ovl|hlp|inf|msc|msp|mst| \
    		          bin|btm|chm|jtd|oft|ops|pcd|prf|scf|sct|shb| \
			  pl|plx|shb|shs|vss|vst|ade|adp|bas|crt|mda|mdb|mde|mdt|mdw|mdz

After, put in acl\_check\_mime:

    #-BLK: Nega links para arquivos com extensoes perigosas
    drop   condition      = ${if <= {$message_size}{SIZ_LinkExecutaveis}{yes}{no}}
           !senders       = wildlsearch;/etc/exim4/lst/fre_lnkexec
           mime_regex     = (?ixs)["']?\\s*(http[s]?|ftp)::\/\/([^"'=>\\s]*)\/([^?"'=>\\s]*)[.](EXT_Executaveis_Link)([?=#]\\w*\\s*)?["']?([<>]|\\s)
           message        = This message contains an link to unwanted file extension mime
           delay          = 45s

2) Spyware images style: (html source: \<img src="http[s]://www.\<spyware
site\>/emailvalidationscript.php?id=youridtovalidate\>
Before, set these in Main Configuration Settings:
SIZ_EmbebedCGI		= 2M

After, put in acl\_check\_mime:

    #-BLK: Nega links com query cgi
    drop   condition      = ${if <= {$message_size}{SIZ_EmbebedCGI}{yes}{no}}
           !senders       = wildlsearch;/etc/exim4/lst/fre_imgcgi
           mime_regex     = (?ixs)src\\s*=\\s*["']?\\s*(http[s]?|ftp)::\/\/([^"'=>\\s]*)\/([^?"'=>\\s]*)[?=#]([^?"'=>\\s]*)\\s*["']?
           message        = This message contains an unwanted Embedded-CGI mime
           delay          = 45s

3) Block False links: (html source: \<a href="http[s]://www.\<hidden and
dangerous\>/redirecttovirus.php\>http[s]://www.\<another inofensive
link\>/\</a\>

Before, set these in Main Configuration Settings:
SIZ_LinkFakeShow	= 2M

After, put in acl\_check\_mime:

    drop   condition      = ${if <= {$message_size}{SIZ_LinkFakeShow}{yes}{no}}
           mime_regex     = (?ixs)href\\s*=\\s*["']?\\s*(http::\/\/)(.+)([\/](.*))?\\s*["']?(\\s*\\w*)+>(\\s*\\w*)+(http::\/\/)(.*)[\/]*(\\s*\\w*)+<
           !mime_regex    = (?ixs)href\\s*=\\s*["']?\\s*(http::\/\/)(.+)([\/](.*))?\\s*["']?(\\s*\\w*)+>(\\s*\\w*)+(http::\/\/)(\\2)[\/]*(\\s*\\w*)+<
           message        = This message displays a link fake, hidden and insecure
           delay          = 45s

4)  Block Singature style virus of outlook:

In system\_filter you put:

    #Virus tipo Assinatura OUTLOOK
    if $message_body matches "(?ixm-s)\
            (?:SCRIPT)(?:[^\"=>]*language=)?(?:3D)?\
                    ([^\>]* \
                     (?:Encode) \
            [^\">]*)"
    then
      logwrite "$tod_log $message_id H=${sender_fullhost} Destinatarios(${recipients_count}): ${recipients} - Remetente: ${sender_address} => Script: $1 - Subject: $header_subject"
      fail text "This message contains an unwanted script encode\n\
                 Script: $1"
      seen finish
    endif

5) Block Object files embeebed (optional, outlook 2k use it):

In system\_filter you put (otional):

    #Objetos Vinculados
    #if $message_body matches "(?ixm-s)\
    #       (?:OBJECT.*classid=)\
    #           (classid=".*"
    #            codebase=
    #                ([^\">]* \
    #       [^\">]*)"
    #then
    #  logwrite "$tod_log $message_id H=${sender_fullhost} Destinatarios(${recipients_count}): ${recipients} - Remetente: ${sender_address} => Objeto: $1 - Subject: $header_subject"
    #  fail text "This message contains an unwanted link for embebbed object\n\
    #             Obj: $1"
    # seen finish
