+++
title = "Server-side mail filtering with Sieve and Dovecot"
date = 2021-09-09
+++

# Adding Sieve filters to Dovecot

Today this blog post is about adding Sieve server-side mail filtering capabilities to Dovecot. This configuration was done on an OpenBSD machine but should work under Linux as well. The configuration shown here enables Sieve filters to be added via the mail client of your choice without ssh'ing to the mail server. Though you want to not enable it or turn it off for security reasons.

Quick summary of what Sieve filters are. A quote from [Sieve.info](http://sieve.info/).
> Sieve ([RFC 5228](http://www.ietf.org/rfc/rfc5228.txt)) is a language for filtering e-mail messages. It is designed to be implementable on either a mail client or mail server. It is meant to be extensible, simple and independent of access protocol, mail architecture and operating system. It is suitable for running on a mail server where users may not be allowed to execute arbitrary programs, such as on black box IMAP servers, since in its basic form it has no variables, loops or ability to shell out to external programs. 

<br></br>

## Install package/ Dovecot configuration

Under OpenBSD the package `dovecot-pigeonhole` adds Sieve mail filter capabilities to Dovecot. Install it.

```sh
$ doas pkg_add dovecot-pigeonhole
```

There is little configuration to be done for basic functionality. Activate the ManageSieve protocol in the configuration file `20-managesieve.conf`. This enables web functionality where you can upload sieve scripts with a mail client over the web. If this is not wanted don't enable it. 

```cfg
protocols = $protocols sieve

service managesieve-login {
    inet_listener sieve
        port = 4190
    }
```

Now enable the plugin for the LMTP protocol, which uses the Sieve filters before it delivers the mail to the mail folders. The configuration file for LMTP settings is `20-lmtp.conf`:

```cfg
protocol lmtp {
    mail_plugins = $mail_plugins sieve
}
```

Check with `doveconf -n | grep protocols` and confirm that the sieve protocol is activated. `doveconf -n` shows all non default settings added to the Dovecot configuration. It should look something like this.

```sh
protocols = imap lmtp sieve
```

The default values for `90-sieve.conf` and `90-sieve-extprograms` should be fine in most use cases. Though you should at least take a quick look at the files.

Reload Dovecot, so the settings take effect. If web functionality is wanted or needed, open port `4190` in your firewall as well.

```sh
$ doas rcctl reload dovecot
```

If you don't want to edit the filter rules via your mail client, in `90-sieve.conf` the location from where the script files are read is defined. By default, it is next to your mailbox in a folder named `sieve`. If the mailbox location is set via `mail_location`, it should be next to it. If the folder doesn't exist yet, create it. Sieve files have the file type ending `.sieve`. If you are unsure about your mailbox location check it with `dovecot -n`.
<br></br>


## Writing Sieve filter rules

Sieve has a really simple filtering syntax. At the top of the file, extensions that are used are included with the keyword `require`. If there is more than a single extension they have to be put in a list defined by square brackets and values separated by commas like so `[argument1, argument2]`.

Example snippet:
```cfg
require ["fileinto", "envelope", "vacation"];
```

The actual rules are matched inside if clauses. To be executed commands are put afterwards inside curly brackets. Sieve matches Strings and provides `":is" ":contains" ":matches"` as comparison operators.

Example snippet:
```cfg
require ["fileinto", "envelope"];

if address :is "from" "annoying-advertisement@company.biz" {
    fileinto "JUNK";
}
```

The snippet above matches only if the mail is sent exactly from that address. If so the mail is moved to the junk folder.

Complete sample Sieve configuration file:

```cfg
require [ "fileinto", "envelope"];

if address :is [ "to","cc" ] ["list@linux-news.org", "list@gnu-news.org"] {
  fileinto "mailing list.linuxnews";
}
```

The extension `fileinto` is needed to refile the mail anywhere except `INBOX`. `Envelope` is required to be able to evaluate mail envelope fields like `sender recipient cc bcc`. For matching more than one string concurrently, lists are needed. Folders in a path are separated by a period.

For more Sieve examples view [these](https://doc.dovecot.org/configuration_manual/sieve/examples/) [links](https://en.wikipedia.org/wiki/Sieve_%28mail_filtering_language%29#Example). Here is a list of Dovecot-pigeonhole supported [Sieve extensions](https://doc.dovecot.org/configuration_manual/sieve/pigeonhole_sieve_interpreter/).
<br></br>

### Additional Sieve documentation/ Troubleshooting

Have a look at the Sieve [wiki](http://sieve.info/).
Here is a good supplementary [Sieve tutorial](https://p5r.uk/blog/2011/sieve-tutorial.html) which goes more in depth concerning the Sieve syntax.

If you want to troubleshoot the auth connection via a GUI mail client you may want to try [claws-mail](https://www.claws-mail.org/), as it has pleasant logging support built in.

Also if you need help or I made a mistake feel free to drop me a [mail](mailto:patrick@folie.dev).
