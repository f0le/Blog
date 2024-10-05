openbsd pkg-readme:
+-------------------------------------------------------------------------------
| Running jitsi on OpenBSD
+-------------------------------------------------------------------------------

A basic configuration guide is provided here which will set up a single node
jitsi-meet instance where anyone can create a conference room and invite others
to join them.
We will assume that the domain of interest is 'example.com' and jitsi is being
hosted in the subdomain 'jitsi.example.com'.

OpenBSD daemons
===============

As jitsi has a lot of moving parts, a concise list of daemons and their
configuration files is presented here for clarity:

1) jvb - (daemon) jitsi videobridge
    * /etc/jvb/jvb.in.sh - default command line parameters and
                                    their values
    * /etc/jvb/jvb.conf - default config file
    * /etc/jvb/sip-communicator.properties - config file for running
                                                      behind a NAT

2) jicofo - (daemon) jitsi conference focus
    * /etc/jicofo/jicofo.in.sh - default command line parameters
                                          and their values
    * /etc/jicofo/jicofo.conf - default config file

3) jitsi-meet - static files for jitsi web frontend
    * /var/www/jitsi-meet/ - default location of files
    * /var/www/jitsi-meet/config.js - default config file

4) nginx - (daemon) web server and reverse proxy
    * /etc/nginx/ - default config files

5) prosody - (daemon) XMPP server used by jitsi
    * /etc/prosody/prosody.cfg.lua - default config file
    * /var/prosody/ - default runtime files

Sample files
============

There is sample file provided for prosody to go along with the default files
provided for jvb and jicofo, located at:
    /usr/local/share/jitsi/prosody.cfg.lua.sample.

Nginx can be used as a reverse proxy, with a configuration for the server
given as follows:

    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        server_name jitsi.example.com;

        ssl_certificate /etc/ssl/jitsi.example.com.crt;
        ssl_certificate_key /etc/ssl/private/jitsi.example.com.key;

        root /jitsi-meet;

        # BOSH
        location = /http-bind {
            proxy_pass      http://127.0.0.1:5280/http-bind;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header Host $http_host;
        }

        ssi on;
        ssi_types application/x-javascript application/javascript;

        location ~ ^/(libs|css|static|images|fonts|lang|sounds|connection_optimization)/(.*)$ {
            add_header 'Access-Control-Allow-Origin' '*';
            alias /jitsi-meet/$1/$2;
        }

        # rooms
        location ~ ^/([a-zA-Z0-9=\?]+)$ {
            rewrite ^/(.*)$ / break;
        }

        # external_api.js must be accessible from the root of the
        # installation for the electron version of Jitsi Meet to work
        location /external_api.js {
            alias /jitsi-meet/libs/external_api.min.js;
        }
    }

Passwords
=========

Throughout the configuration, the following passwords should be generated as
they will be needed in the configuration files:

    ${JAVA_TS_PASS}
    ${JVB_COMP_PASS}
    ${FOCUS_COMP_PASS}

pf.conf
=======

The default configuration uses the following ports:

    * nginx: TCP 80, 443
    * prosody: TCP 5000, 5222, 5269, 5280, 5281, 5347, 5582
    * jicofo: TCP 8888
    * jvb: TCP 8080, UDP 10000

Only a few ports, TCP 80, 443 and UDP 10000, are to be exposed to the
network, the other ports are used for internal communication between jicofo,
jvb and prosody.
A possible set of pf.conf rules that can be used is:

    pass in on egress to (egress) tcp port { 80 443 }
    pass in on egress to (egress) udp port 10000

/etc/hosts configuration
========================

Jitsi needs two subdomains, 'auth.jitsi.example.com' and 'jitsi.example.com',
configured as part of the setup, of which only 'jitsi.example.com' is
exposed outside the local network.

They are accessed by the jicofo, jvb and prosody daemons as part of their
internal communication. The simplest way to make them resolvable to localhost
is to add them in the /etc/hosts file -

    127.0.0.1 localhost jitsi jitsi.example.com auth.jitsi auth.jitsi.example.com
    ::1       localhost jitsi jitsi.example.com auth.jitsi auth.jitsi.example.com

Nginx configuration
===================

Jitsi uses webrtc which mandates the use of https. The sample nginx config file
should be updated to use the proper TLS certificates, which can be obtained
by acme-client(1). These are also going to be used by prosody.

Prosody configuration
=====================

In the sample prosody configuration file, replace the domain and the password
placeholders with the passwords chosen above.

In the section for the domain 'jitsi.example.com' the certificates obtained in
the previous step should be used.

Prosody also hosts the internal domain 'auth.jitsi.example.com' and can use
self signed TLS certificates for this.
They should be generated using the following command:

    # prosodyctl cert generate auth.jitsi.example.com

The certificates will be stored in:
    /var/prosody/auth.jitsi.example.com.{crt,key}.

These certificates also need to be shared with jicofo and jvb by adding them
to a Java certificate truststore /etc/ssl/jitsi.store.

    # $(javaPathHelper -h jicofo)/bin/keytool -import -alias prosody \
      -file /var/prosody/auth.jitsi.example.com \
      -keystore /etc/ssl/jitsi.store -storepass ${JAVA_TS_PASS}

Prosody needs two plugins to be added to the setup which can be achieved by:

    # prosodyctl install --server=https://modules.prosody.im/rocks/ \
        mod_client_proxy
    # prosodyctl install --server=https://modules.prosody.im/rocks/ \
        mod_roster_command

The 'focus' user for prosody should also be registered via the command line:

    # prosodyctl register focus auth.jitsi.example.com ${FOCUS_COMP_PASS}
    # prosodyctl mod_roster_command subscribe focus.jitsi.example.com \
        focus@auth.jitsi.example.com

JVB and jicofo configuration
============================

The default configuration files for jvb and jicofo only need the domain and
password fields to be updated.
The jicofo daemon needs to be provided the host name:

    # rcctl set jicofo flags "--host=jitsi.example.com"

SIP configuration
=================

If the jitsi server is behind a NAT, such as when hosting from an internal
homeserver, the config file /etc/jvb/sip-communicator.properties
should be updated to include the public and NAT local addresses of the setup.
The ${LOCAL_ADDRESS} should be the internal IP address assigned on the LAN
network and the ${PUBLIC_ADDRESS} should be the one used by peers outside
the LAN to reach the setup.

Jitsi-meet configuration
========================

The relevant parts of the web configuration file at
'/var/www/jitsi-meet/config.js' that need to be updated, and
uncommented if needed, are provided here:

    var config = {
      hosts: {
        domain: 'jitsi.example.com',
        muc: 'conference.jitsi.example.com'
      },

      bosh: '//jitsi.example.com/http-bind',
      useTurnUdp: false,
      enableWelcomePage: true,
      prejoinConfig: {
        enabled: true,
        hideExtraJoinButtons: ['no-audio', 'by-phone']
      },
      p2p: {
        stunServers: [ { urls: 'stun:meet-jit-si-turnrelay.jitsi.net:443' } ]
      }
    }

Spinning up the daemons
=======================

The daemons needs to be started in the order given:

    # rcctl enable nginx prosody jicofo jvb
    # rcctl order nginx prosody jicofo jvb
    # rcctl start nginx prosody jicofo jvb

The setup can be tested by visiting the site at https://jitsi.example.com.

Additional upstream documentation
=================================

Further steps to configure the setup can be found in the upstream
documentation at https://jitsi.github.io/handbook/.

change pf:
allow incoming tcp port on 10000 for jvb 

change /etc/hosts for subdomains
 127.0.0.1 localhost jitsi jitsi.example.com auth.jitsi auth.jitsi.example.com
    ::1       localhost jitsi jitsi.example.com auth.jitsi auth.jitsi.example.com


