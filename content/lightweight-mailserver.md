+++
title = "Lightweight mail server"
date = 2020-07-27

+++

# Lightweight personal mail server

This is a write-up of how to install and configure a lightweight personal mail server running on [OpenBSD](https://www.openbsd.org/). Installing OpenBSD is out if scope of this article however. If you need a primer on OpenBSD check out this [link](https://www.openbsdjumpstart.org/).

The Software used is [OpenSMTPD](https://www.opensmtpd.org/) for sending and receiving mail, [Dovecot](https://www.dovecot.org/) for handling IMAP, [Rspamd](https://rspamd.com/) for handling spam, as well as signing outgoing mails and the domain provider and/or hoster of your choice.
Make sure your hoster has a setting for a reverse IP entry tho.
For better performance you can use [Redis](https://redis.io/) as a caching server. It requires little to no configuration.

Some people reading this may find the term "lightweight" to be somewhat a joke, but running a mail server is not as hard as most people think. Especially with a lightweight software approach and the fabulous software that is OpenSMTPD which will handle a lot of the heavy lifting.
This blog post is in large part inspired by a how-to written by one of the main contributors of said OpenSMTPD. Check out his blog [here](https://poolp.org/).

The mail-server will use a virtualised user management so you don't have to add a user per mail box on your OS. The user virtualmail will receive all the mails and access will be managed by Dovecot over IMAP.
A flat file will handle the user management so you can employ your own script if you need to add/modify/remove a mail user account or an alias.

In this blog post I won't explain basic concepts of e-mail, only reference some links for further reading in the respective paragraphs. This post is intended to be more of a hands on approach to a sane and scalable lightweight mail server configuration.

<br></br>

## DNS and rDNS

The DNS settings are important for the mail server, especially FCrDNS. It improves the reputation of the mail server with the big blacklist providers and the mail corporations.
Make sure your server has a correct reverse DNS entry. Most spammers use compromised machines on residential connections and these connections lack a reverse DNS address. That makes it a good indicator of a bogus mail servers for blacklist providers. So a lack of correctly set up reverse DNS is a prime reason of rejected mails.

Your mail server has to be reachable from the internet. That is done by editing your DNS zone. Replace the placeholder domains `mail.domain.com,www.domain.com,domain.com` with your domain name and `your-ipv4-address` with your IP address. Take care of the punctuation. IPv6 is optional.

```config
www.domain.com.      CNAME   domain.com.
domain.com.          A       your-ipv4-address
mail.domain.com.     A       your-ipv4-address 
mail.domain.com.     AAAA    your-ipv6-address 
domain.com.          MX 0    mail.domain.com.
```

When you update these values, bear in mind the propagation delay.

Now check that the MX entries are correct:

```sh
$ dig -t MX mail.domain.com +short
0 mail.domain.com.
```
Check that your domain resolves to the correct IP addresses:

```sh
$ host mail.domain.com
mail.domain.com has address your-ipv4-address
mail.domain.com has IPv6 address your-ipv6-address
```

Set the rDNS record somewhere in the control panel of your hoster.
Check the reverse DNS afterwards:

```sh
$ host your-ipv4-address
your-ipv4-address.in-addr.arpa domain name pointer mail.domain.com.
$ host your-ipv6-address
your-ipv6-address.ip6.arpa domain name pointer mail.domain.com.
```

```sh
$ host mail.domain.com
mail.domain.com has address your-ipv4-address
mail.domain.com has IPv6 address your-ipv6-address
```
If both match each other you have successfully set up your DNS and rDNS. Also called FCrDNS. Read more about it [here](https://en.wikipedia.org/wiki/Forward-confirmed_reverse_DNS).

### SPF

The SPF domain entry is pretty simple, just add a TXT record to your DNS zone.
The following configuration announces that only the server defined in the MX entry is allowed to send mail for `domain.com`.

```config
domain.com. IN TXT "v=spf mx -all"
```

### DKIM

Generate a DKIM key. Make sure to at least generate a key with 1024 bits, 2048 is better.

```sh
$ doas openssl genrsa -out /etc/mail/mail.domain.com.key 2048
$ doas openssl rsa -in /etc/mail/mail.domain.com.key -pubout -out /etc/mail/mail.domain.com.pubkey
$ cat /etc/mail/mail.domain.com.pubkey
```

Now add an entry for your key in your DNS zone. Exchange `yourkey` with your generated public RSA key. And replace dkimselector with a name of your choosing. If you generated a 2048 bit key and your domain registrar doesn't support DKIM DNS entries you have to split your key in two, because of the character limitation of TXT records.
Most of the time that is being done by enclosing the public DKIM entry into parentheses and enclosing the split parts in double quotes. This may differ with your registrar. Examples below.

This is a  1024 bit key record:
```config
dkimselector._domainkey.domain.com. IN TXT "v=DKIM1;k=rsa;h=sha256;p=yourkey"
```

A 2048 bit DKIM DNS record should look similar to this:
```config
dkimselector._domainkey.domain.com. IN DKIM "v=DKIM1;k=rsa;h=sha256;p=yourkey"
```

A 2048 bit key split in half may look like this:
```config
dkimselector._domainkey.domain.com. IN TXT ("v=DKIM1;k=rsa;h=sha256;p=firstpartofyourkey"
```

```config
dkimselector._domainkey.domain.com. IN TXT "secondpartofyourkey")
```

If you have problems with this, this [website](https://www.mailhardener.com/tools/dns-record-splitter) seems helpful. You can also check the validity of your records there. To check if your messages are correctly signed test [here](https://dkimvalidator.com/).

Set permissions on the DKIM keys, so the Rspamd daemon can access the files.

```sh
$ doas chown _rspamd:_rspamd /etc/mail/mail.domain.com.key
$ doas chown _rspamd:_rspamd /etc/mail/mail.domain.com.pubkey
$ doas chmod 0600 /etc/mail/mail.domain.com.key
$ doas chmod 0644 /etc/mail/mail.domain.com.pubkey
```

### DMARC

Add a DMARC entry in your DNS zone. This entry is mostly cosmetic, it tells another mail server to send a DMARC report to `postmaster@domain.com`. Also it informs what `domain.com` will do with mails failing the DMARC checks, namely nothing. It is a good enough default configuration for small mail servers.

```config
_dmarc.domain.com. IN TXT "v=DMARC1;p=none;pct=100;rua=mailto:postmaster@domain.com;"
```

Now that we are done with the DNS configuration we can install the packages and configure our mail server.

## Installation

Install needed packages on OpenBSD:

```sh
$ doas pkg_add opensmtpd-extras opensmtpd-filter-senderscore opensmtpd-filter-rspamd dovecot dovecot-pigeonhole rspamd redis
```

The `opensmtpd-filter-senderscore` filter rejects mail based on a score depending on some factors, whereas the `dovecot-pigeonhole` package is used to train Rspamd for spam mail detection. Both are optional but nice to have.

If you want to use the same file for OpenSMTPD and Dovecot user authentication `opensmtpd-extras` is needed.

The `redis` package is used to improve the performance of the mail server. 

### Add user 

Add a user which is only used for mail reception:

```sh
$ doas useradd -M virtualmail
```

### Modify firewall rules

Configuring pf is unfortunately out of scope if this article, the OpenBSD User guide is available [here](https://www.openbsd.org/faq/pf/). Expect a post about the configuration of pf and the differences between pf and iptables in the future.
The submission port as well as the imaps and smtps ports are needed in the firewall.

### Post installation configuration

This is OpenBSD specific. Follow the instructions of the post install readme. For larger servers increase these values. In short:

```sh
$ doas vim /etc/login.conf
```

```config
dovecot:\
    :openfiles-cur=1024:\
    :openfiles-max=2048:\
    :tc=daemon:
```

If needed rebuild the login.conf.db:

```sh
$ doas [ -f /etc/login.conf.db ] && cap_mkdb /etc/login.conf
```

### Create mail directory

```sh
$ doas mkdir -p /var/virtualmail
```

Set permissions:
```sh
$ doas chown virtualmailuser:virtualmailuser /var/virtualmail
```

## Configuration

### SSL Certificates

If you don't yet have a SSL Certificate you can get one from Let's Encrypt. There are many guides out there on how to do that, so I won't go into detail here. OpenBSD has a tool `acme-client` to automatically fetch SSL Certificate from Let's Encrypt, a webserver is needed also. See `/etc/example/httpd.conf` and `man acme-client` for details.

### Virtualised User management

The benefit of a flexible configuration like this, is the possibility to add/remove/modify mail accounts or mail aliases on the fly. Mail aliases can be added in the `/etc/mail/virtualusers`. User accounts can be added via `/etc/mail/credentials`, make sure to also add the corresponding entry in `/etc/mail/virtualusers`. All mail will be delivered to the user `virtualmail` created earlier. The configuration for the location and folder structure will be done later in the dovecot paragraph.

We need several files to do this. First define the domains which are being served by the mail server.

```sh
$ doas vim /etc/mail/domains
```

```config
domain.com
*.domain.com
```

This file maps virtual mail addresses on the left to addresses (virtual or not) on the right. Last recipient is the user which receives all the mail. 

```sh
$ doas vim /etc/mail/virtualusers
```

```config
abuse@domain.com        info@domain.com
postmaster@domain.com   info@domain.com
webmaster@domain.com    info@domain.com

info@domain.com         username@domain.com
username@domain.com     virtualmail
```

In this configuration we will use one file for the authentication management for Dovecot and OpenSMTPD. The file format for this file will be in a /etc/passwd style format. So make sure you have `opensmtpd-extras` installed as it is needed to use a /etc/passwd formatted file for OpenSMTPD. See `man table-passwd` for more information. The hash has to be generated with `smtpctl encrypt`. Replace `hash` with your hashed passwords for the users.

Take care to add as many `:` as in the example, as they are needed for the right formatting.
```sh
$ doas vim /etc/mail/passwords
```

```config
info@domain.com:hash:::::::
username@domain.com:hash:::::::
```

As an example; `smtpctl encrypt foo` results in the following hash on my machine: `$2b$09$nDJiOSWPJLl4bITY5XTvgekcM82Gn4RLpk9UFwkmns2HbSYJY3/4a`\
The hashing function used by default is bcrypt.

These are the mail aliases for the local daemons and users. We will be sending all the mail generated by programs to our normal user on the system. This user will have access to the mail via a local mailbox and the simple `mail` command. So you could alias all the mail to root and root's mail to your user account.

```sh
$ doas vim /etc/mail/aliases
```

```config
#
#	$OpenBSD: aliases,v 1.67 2019/01/26 10:58:05 florian Exp $
#
#  Aliases in this file will NOT be expanded in the header from
#  Mail, but WILL be visible over networks or from /usr/libexec/mail.local.
#
#	>>>>>>>>>>	The program "newaliases" must be run after
#	>> NOTE >>	this file is updated for any changes to
#	>>>>>>>>>>	show through to smtpd.
#

# Basic system aliases -- these MUST be present
MAILER-DAEMON: postmaster
#postmaster: root

# General redirections for important pseudo accounts
daemon:	root
ftp-bugs: root
operator: root
www:	root

# Redirections for pseudo accounts that should not receive mail
_bgpd: /dev/null
_dhcp: /dev/null
_dpb: /dev/null
_dvmrpd: /dev/null
_eigrpd: /dev/null
_file: /dev/null
_fingerd: /dev/null
_ftp: /dev/null
_hostapd: /dev/null
_identd: /dev/null
_iked: /dev/null
_isakmpd: /dev/null
_iscsid: /dev/null
_ldapd: /dev/null
_ldpd: /dev/null
_mopd: /dev/null
_nsd: /dev/null
_ntp: /dev/null
_ospfd: /dev/null
_ospf6d: /dev/null
_pbuild: /dev/null
_pfetch: /dev/null
_pflogd: /dev/null
_ping: /dev/null
_pkgfetch: /dev/null
_pkguntar: /dev/null
_portmap: /dev/null
_ppp: /dev/null
_rad: /dev/null
_radiusd: /dev/null
_rbootd: /dev/null
_relayd: /dev/null
_rebound: /dev/null
_ripd: /dev/null
_rstatd: /dev/null
_rusersd: /dev/null
_rwalld: /dev/null
_smtpd: /dev/null
_smtpq: /dev/null
_sndio: /dev/null
_snmpd: /dev/null
_spamd: /dev/null
_switchd: /dev/null
_syslogd: /dev/null
_tcpdump: /dev/null
_traceroute: /dev/null
_tftpd: /dev/null
_vmd: /dev/null
_x11:   /dev/null
_ypldap: /dev/null
bin:	/dev/null
build:	/dev/null
nobody:	/dev/null
_tftp_proxy: /dev/null
_ftp_proxy: /dev/null
_sndiop: /dev/null
_syspatch: /dev/null
_slaacd: /dev/null
sshd:   /dev/null

# Well-known aliases -- these should be filled in!
root:	joe
# manager:
# dumper:

# RFC 2142: NETWORK OPERATIONS MAILBOX NAMES
#abuse:		root
# noc:		root
#security:	root

# RFC 2142: SUPPORT MAILBOX NAMES FOR SPECIFIC INTERNET SERVICES
# hostmaster:	root
# usenet:	root
# news:		usenet
# webmaster:	root
# ftp:		root
```

Run `newaliases` after you made changes to rebuild `aliases.db` if needed.

### OpenSMTPD

The configuration file is using the syntax introduced in 6.6.0 version so it will not work with earlier versions.

The server won't be running an open relay with this configuration as you will often see in other tutorials on the web.
Local users will still be able to check their mail with the `mail` command on OpenBSD. Now configure the smtpd server in `/etc/mail/smtpd.conf`:

```config
#Define Certificates
pki mail.domain.com cert "/etc/ssl/live/domain.com/fullchain.pem"
pki mail.domain.com key "/etc/ssl/live/domain.com/privkey.pem"

#Define keys for Sender Rewriting Scheme
srs key "yoursecretkey"
srs key backup "yoursecretbackupkey"

#Define Tables
table aliases file:/etc/mail/aliases
table domains file:/etc/mail/domains
table passwords passwd:/etc/mail/passwords
table virtualusers file:/etc/mail/virtualusers

#Define Filters
filter check_dyndns phase connect match rdns regex { '.*\.dyn\..*', '.*\.dsl\..*' } disconnect "550 no residential connections"
filter check_rdns phase connect match !rdns disconnect "550 no rDNS"
filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS"
filter senderscore proc-exec "filter-senderscore -blockBelow 10 -junkBelow 70 -slowFactor 5000"
filter rspamd proc-exec "filter-rspamd"

#Define ports to listen on
listen on all tls pki mail.domain.com filter { check_dyndns, check_rdns, check_fcrdns, senderscore, rspamd}
listen on all port submission tls-require pki mail.domain.com auth <passwords> filter rspamd no-dsn mask-src

#Define Actions
action "localmail" mbox alias <aliases>
action "domainmail" lmtp "/var/dovecot/lmtp" rcpt-to virtual <virtualusers>
action "outbound" relay srs helo mail.domain.com

#Define Rules, matching sequentially
#Inbound
match from local for local action "localmail"
match from any for domain <domains> action "domainmail" 

#Outbound
match from local for any action "outbound"
match auth from any for any action "outbound"
```

Explanation of the configuration values:

<b>Certificates</b>

We define the SSL Certificates the server uses. The certificate chain and your private key.

<b>Sender Rewriting Scheme</b>

We define the keys used for SRS. This is important, because it makes sure forwarding mail doesn't break SPF. More info on [SRS](https://en.wikipedia.org/wiki/Sender_Rewriting_Scheme). To generate the keys this should suffice:
`openssl rand -base64 32 | cut -b -30`

<b>Tables</b>

We define the tables we created earlier. The domains, local aliases, virtual users and user credentials. The actual user accounts are defined in `passwords`, the `virtualusers` file merely defines the mail aliases.

<b>Filters</b>

Incoming connections to our SMTP server are being filtered by these rules.\
The first rule checks for a regex in the rDNS which matches residential connections.\
The second checks if there exists a rDNS entry.\
The third checks for FCrDNS.\
The fourth executes the process `filter-senderscore` which filters mail based on a score determined by the program. The configuration for the filter can be adjusted in /etc/rspamd/actions.conf\
The last executes the `filter-rspamd` to check for spam.

<b>Listening interfaces/ports</b>

The server listening directive.\
The first line defines to only open encrypted connections and use the filters defined earlier.\
The second line tells the server to listen on all interfaces and on the `submission` port as well as require TLS.\
It then checks authentication against the defined table, then filters through Rspamd. It disables the DSN extension(delivery status extension) to prevent the sender spying on you and uses `mask-src` which omits the "from" part when prepending “Received” headers.\

<b>Actions</b>

What the server does with the mail it receives is defined here.\
The first action delivers mail which is matched by "localmail" to the users mailbox in mbox format defined in the alias table.\
The second action delivers mail which is destined for the domain addresses to the corresponding user defined in the virtualusers table via `lmtp "/var/dovecot/lmtp" rcpt-to`. LMTP is the local mail transport protocol.\
The third action sends mail which is matching the "outbound" rule, using SRS to our SMTP-server for outbound delivery.

<b>Rules</b>

How the server handles the mail is defined here.
The first inbound rule defines that mail arriving from the local system should be matched by the "localmail" rule.\
The second inbound rule defines that mail arriving from anywhere for one of the domains matched in "domains" should be matched by the "domainmail" rule.\

The first outbound rule defines that mail coming from the local system for any recipient except locals should be matched by the "outbound" rule.\
The second outbound rule defines that mail coming from anywhere for any recipient except locals should be matched by the "outbound" rule as well.\

Start and enable the service.
```sh
$ doas rcctl start smtpd
$ doas rcctl enable smtpd
```

<b>Troubleshooting</b>

The documentation on OpenBSD is excellent, `man smtpd` and `man smtpd.conf` are recommended. 
If you need help with the configuration syntax check out the [blog](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/) of one of the main contributors on OpenSMTPD. See [here](https://prefetch.eu/blog/2020/email-server/) for another write-up on mail server configuration.

To troubleshoot problems the server can be started in the foreground with subsystem tracing.\
E.g. for debugging the smtp subsystem: `smtpd -dv -T smtp`. Change the trace option as needed.

To test the connection to your server you can use:`$ sudo openssl s_client -host mail.domain.com -port 587 -starttls smtp`

### Dovecot

Now on to the configuration of the IMAP server.

Dovecot's configuration is divided into a whole directory with different configuration files. We will only be needing to edit few of them. I will post whole uncommented configuration files here, so to avoid confusion. 

Main settings file:

```sh
$ doas vim /etc/dovecot/dovecot.conf
```

```config
protocols = imap lmtp
listen = *
dict {
}
!include conf.d/*.conf

!include_try local.conf
```

General settings:
```sh
$ doas vim /etc/dovecot/conf.d/10-master.conf
```

```config
service imap-login {
    inet_listener imap {
    port = 0
    }
    inet_listener imaps {
    port = 993
    ssl = yes
    }
}
service pop3-login {
}
service submission-login {
}
service lmtp {
    user = virtualmail
    unix_listener lmtp {
    }
}
service imap {
}
service pop3 {
}
service submission {
}
service auth {
    uix_listener auth-userdb {
    }
}
service auth-worker {
}
service dict {
    unix_listener dict {
    }
}
```

Auth-configuration:
```sh
$ doas vim /etc/dovecot/conf.d/10-auth.conf
```

```config
auth_default_realm = domain.com
auth_mechanisms = plain
!include auth-passwdfile.conf.ext
```

Mail-location configuration:
```sh
$ doas vim /etc/dovecot/conf.d/10-mail.conf
```

```config
mail_location = maildir:/var/virtualmail/%d/%n/Maildir
namespace inbox {
    inbox = yes
}
mmap_disable = yes
first_valid_uid = 1000
mail_plugin_dir = /usr/local/lib/dovecot
protocol !indexer-worker {
}
mbox_write_locks = fcntl
```

Logging-configuration:
```sh
$ doas vim /etc/dovecot/conf.d/10-logging.conf
```

```config
log_path = syslog
debug_log_path = syslog
syslog_facility = mail
auth_verbose = no
auth_debug = no
auth_debug_passwords = no
mail_debug = no
mail_debug = no
verbose_ssl = no
plugin {
    mail_log_events = delete undelete expunge copy mailbox_delete mailbox_rename
    mail_log_fields = uid box msgid size
}
log_timestamp = "%b %d %H:%M:%S "
mail_log_prefix = "%s(%u)<%{pid}><%{session}>: "
deliver_log_format = msgid=%m: %$ %{storage_id} %{to_envelope}
```
For troubleshooting purposes change the debug values above to yes.
Dovecot configuration manual is available [here](https://doc.dovecot.org/configuration_manual/).

SSL-configuration:
```sh
$ doas vim /etc/dovecot/conf.d/10-ssl.conf
```

```config
ssl = required
ssl_cert = </etc/letsencrypt/live/domain.com/fullchain.pem
ssl_key = </etc/letsencrypt/olive/domain.com/privkey.pem
ssl_min_protocol = TLSv1.2
ssl_prefer_server_ciphers = yes
```

OpenBSD uses sane defaults for cryptography ciphers so no changes are needed.

Auth-configuration:\
Default scheme for the hashing used by `smtpctl encrypt` is `BLF-CRYPT`(bcrypt). Change the `home` variable under `userdb` to how the folder structure should look like.
```sh
$ doas vim /etc/dovecot/conf.d/auth-passwdfile.conf.ext
```

```config
passdb {
    args = scheme=BLF-CRYPT username_format=%n /etc/mail/credentials
    driver = passwd-file
}

userdb {
    args = uid=virtualmail gid=virtualmail username_format=%n home=/var/virtualmail/%d/%n
    driver=static
}
```

The following configurations are optional but nice to have if you want to use spam recognition training.

IMAP configuration:
```sh
$ doas vim /etc/dovecot/conf.d/20-imap.conf
```

```config
protocol imap {
    mail_plugins = $mail_plugins imap_sieve
}
```

Sieve plugin configuration:
```sh
$ doas vim /etc/dovecot/conf.d/90-plugin.conf
```

```config
plugin {
  sieve_plugins = sieve_imapsieve sieve_extprograms
  sieve_global_extensions = +vnd.dovecot.pipe +vnd.dovecot.environment

  imapsieve_mailbox1_name = Junk
  imapsieve_mailbox1_causes = COPY APPEND
  imapsieve_mailbox1_before = file:/usr/local/lib/dovecot/sieve/report-spam.sieve

  imapsieve_mailbox2_name = *
  imapsieve_mailbox2_from = Junk
  imapsieve_mailbox2_causes = COPY
  imapsieve_mailbox2_before = file:/usr/local/lib/dovecot/sieve/report-ham.sieve

  imapsieve_mailbox3_name = Inbox
  imapsieve_mailbox3_causes = APPEND
  imapsieve_mailbox3_before = file:/usr/local/lib/dovecot/sieve/report-ham.sieve

  sieve_pipe_bin_dir = /usr/local/lib/dovecot/sieve
}
```

```sh
$ cd /usr/local/lib/dovecot/sieve
$ doas sievec report-ham.sieve
$ doas sievec report-spam.sieve
$ doas chmod 755 sa-learn-ham.sh
$ doas chmod 755 sa-learn-spam.sh
```

Start and enable the service.
```sh
$ doas rcctl start dovecot
$ doas rcctl enable dovecot
```

<b> Troubleshooting</b>

For more information on the configuration parameters, the dovecot configuration manual is available [here](https://doc.dovecot.org/configuration_manual/). `Doveadm` is Dovecot's administration tool. It can be used to debug mailboxes as well as authentication issues.

### Rspamd

Rspamd is there to filter your mail for spam and to sign outgoing messages with a digital signature. Namely DKIM, for more info see e.g. [Wikipedia](https://en.wikipedia.org/wiki/DomainKeys_Identified_Mail). Edit the path to point to the path of your generated secret RSA key and change `dkimselector` to match the one you used in your DNS zone.

```sh
$ doas vim /etc/rspamd/local.d/dkim.conf
```

```config
allow_username_mismatch = true;
domain {
        domain.com {
                path = "/etc/mail/mail.domain.com.key";
                selector = "dkimselector";
        }
}
```

Start and enable the service.
```sh
$ doas rcctl start rspamd
$ doas rcctl enable rspamd
```

### Redis

Redis works fine without any configuration. Bear in mind that the default configuration uses passwordless access to Redis. If you don't like this you have to change it. More information [here](https://redis.io/topics/security).
Start and enable the service.

```sh
$ doas rcctl start redis
$ doas rcctl enable redis
```

## Testing

Now that everything is running you should be able to test your setup by logging into your account via IMAP and try to send and receive mail. Login should be possible with username and password for the respective fields in your mail client.

If everything is working, Congratulations! You now have a personal working "lightweight" mail server. If you have questions, need help or spotted a mistake drop me a [mail](mailto:patrick@folie.dev). My PGP key is available  [here](https://folie.dev/patrick@folie.dev.asc).
