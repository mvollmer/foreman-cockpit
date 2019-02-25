# Integrating Foreman and Cockpit

There is work in progress to integrate Cockpit into Foreman.
Eventually, setting this up should be trivial but until it's all
wrapped up and shipped -- it's a bit tricky.

## Overview

On the surface, you will get a new "Web Console" button on the page for
a host, right next to "Schedule Remote Job".  Clicking on the button
will open the Web Console (aka Cockpit) for that host.

There are three levels of sophistication and seamlessness:

 1) The button can simple direct the browser to port 9090 on the host.

 2) The button can direct the browser to a specially configured
    instance of Cockpit.  That instance is configured to then connect
    to the actual target host via SSH, using the relevant SmartProxy
    and the appropriate REX credentials for that host.

 3) The specially configured Cockpit instance can be served from the
    same origin as the Foreman UI itself, via reverse proxying of the
    Cockpit webserver.

For 1), the browser needs to be able to connect to the target host
directly, and the target host needs to have the Cockpit webserver
installed and enabled.  The user will land on Cockpits login page and
will have to provide appropriate credentials.

For 2), the browser only needs to be able to connect to the one
special Cockpit instance.  If remote execution credentials are setup
for the target host, the Cockpit login page can be skipped.  The
firewall still needs to be opened for the Cockpit instance.

For 3), the Cockpit traffic uses the exact same host, port, and TLS
certificates as Foreman itself.  This requires that both Cockpit and
Foreman are behind the same reverse proxy.

We are of course aiming for level 3, but lower levels should also be
possible to configure, if only to make hacking on this stuff easier.

## Getting the code

As a prerequisite, you should be running Foreman from source.  [ I
might write down how I did that, if there is interest. ]

Then add the following:

 - Stock Foreman
   https://github.com/theforeman/foreman

 - REX plugin for Foreman with lots of extra sauce
   https://github.com/theforeman/foreman_remote_execution/pull/397
   https://github.com/theforeman/foreman_remote_execution/pull/408
   https://github.com/theforeman/foreman_remote_execution/pull/409

   Here is a branch with all these PRs combined, but it might not
   always be up-to-date:
   https://github.com/mvollmer/foreman_remote_execution/tree/seamless-cockpit-combined

 - SmartProxy with Puma
   https://github.com/theforeman/smart-proxy/pull/623

 - REX plugin for the SmartProxy with a bit of extra sauce
   https://github.com/theforeman/smart_proxy_remote_execution_ssh/pull/40

Even if you have it all setup correctly, you wont see the "Web
Console" button without further configuration.

## Level 1

Set the remote_execution_cockpit_href setting to
"https://%{host}:9090" via the Foreman UI at

    Administer / Settings / RemoteExecution / Cockpit URL.

Now you should have a "Web Console" button on the host page that opens
plain Cockpit for a host.

For level 1, you don't need the SmartProxy or a working REX
configuration.  You _do_ need the special sauce for the REX plugin
(see above), however, so don't skip on that.

## Level 2

[ The following right now needs to happen on the machine that serves
  the Foreman UI and Foreman needs to listen on port 3000 on
  localhost. This restriction will be lifted.
]

Install this config file

```
# /etc/foreman-cockpit-direct/cockpit/cockpit.conf
[Webservice]
LoginTitle = Foreman Cockpit Direct

[Bearer]
Action = remote-login-ssh

[SSH-Login]
command = /usr/libexec/foreman-cockpit-session

[OAuth]
Url = <FOREMAN_URL>/api/cockpit_redirect
```

You need to replace "<FOREMAN_URL>" with the URL of your Foreman server.

Copy (or link) the Foreman session helper to the expected location:

```[shell]
# ln -s .../foreman_remote_execution/tools/foreman-cockpit-session /usr/libexec
```

Install this systemd unit file:

```
# foreman-cockpit-direct.service
[Unit]
Description=Foreman Cockpit Direct Web Service

[Service]
Environment=XDG_CONFIG_DIRS=/etc/foreman-cockpit-direct/
ExecStartPre=+/usr/sbin/remotectl certificate --ensure --user=root --group=cockpit-ws --selinux-type=etc_t
ExecStart=/usr/libexec/cockpit-ws --port 9998
User=cockpit-ws
Group=cockpit-ws
```

Create a necessary directory, open the firewall, and enable the service:

```[shell]
# mkdir -p /etc/foreman-cockpit-direct/cockpit/ws-certs.d
# firewall-cmd --add-port tcp/9998
# firewall-cmd --add-port tcp/9998 --permanent
# systemctl daemon-reload
# systemctl enable --now foreman-cockpit-direct
```

Set the remote_execution_cockpit_href setting to
"https://<FOREMAN_HOST>:9998/=%{host}".  You need to replace
"<FOREMAN_HOST>" with the name or address URL of the Foreman server.

## Level 3

[ The following right now needs to happen on the machine that serves
  the Foreman UI and Foreman needs to listen on port 3000 on
  localhost. This restriction will be lifted.
]

This is quite similar to Level 2 above.

Install this config file

```
# /etc/foreman-cockpit/cockpit/cockpit.conf
[Webservice]
LoginTitle = Foreman Cockpit
UrlRoot = /webcon/

[Bearer]
Action = remote-login-ssh

[SSH-Login]
command = /usr/libexec/foreman-cockpit-session

[OAuth]
Url = /api/cockpit_redirect
```

Copy (or link) the Foreman session helper to the expected location:

```[shell]
# ln -s .../foreman_remote_execution/tools/foreman-cockpit-session /usr/libexec
```

Install this config file for httpd and adjust as necessary:

```
# /etc/httpd/conf.d/cockpit.conf
<VirtualHost *:80>
  ServerName mvo.dev.lan

  ProxyPreserveHost On
  ProxyRequests Off

  RewriteEngine On
  RewriteCond %{HTTP:Upgrade} =websocket [NC]
  RewriteRule /webcon/(.*)           ws://127.0.0.1:9999/webcon/$1 [P]
  RewriteCond %{HTTP:Upgrade} !=websocket [NC]
  RewriteRule /webcon/(.*)           http://127.0.0.1:9999/webcon/$1 [P]

  ProxyPass / <FOREMAN_URL>

</VirtualHost>
```

Install this systemd unit file:

```
# foreman-cockpit.service
[Unit]
Description=Foreman Cockpit Web Service

[Service]
Environment=XDG_CONFIG_DIRS=/etc/foreman-cockpit/
ExecStart=/usr/libexec/cockpit-ws --no-tls --address 127.0.0.1 --port 9999
User=cockpit-ws
Group=cockpit-ws
```

Create a necessary directory, reload httpd, and enable the service:

```[shell]
# systemctl restart httpd
# systemctl daemon-reload
# systemctl enable --now foreman-cockpit
```

Set the remote_execution_cockpit_href setting to "/webcon/=%{host}".

## Open issues

- foreman-cockpit-session talks to localhost:3000, that needs to be configurable.
- level 3 doesn't do tls
- do multiple tabs in one browser for multiple hosts actually work?
