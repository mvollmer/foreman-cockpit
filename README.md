# Integrating Foreman and Cockpit

There is work in progress to integrate Cockpit into Foreman.
Eventually, setting this up should be trivial but until it's all
wrapped up and shipped -- it's complicated.

The following instructions will set up a single new virtual machine
that runs Foreman and you can seamlessly open Cockpit from Foreman for
that same virtual machine.

Current issues:

 - SELinux needs to be switched off.

 - Communication between cockpit-ws and foreman-proxy can not use
   https.

## Setting up the virtual machine

Create a new Centos 7 virtual machine with 4GiB RAM and 10GiB disk
from this:

    http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso

In the installer, make sure that Network is switched on.  Also in the
installer, set the hostname of the new machine to "foreman.devel.lan".

Make sure that reverse DNS lookup gives consistent results inside the
virtual machine.  One easy way is to add a line like this to
`/etc/hosts` on the virtual machine itself:

```
192.168.100.109 foreman.demo.lan foreman
```

Make sure to use the real IP address of the virtual machine, of
course.

Make sure that the new virtual machine can also be reached via its
"foreman.demo.lan" name from the outside.  Again, a easy way is to add
a line like above to `/etc/hosts` on the machine that will run the
browser.

Switch off SELinux permanently:

```
# setenforce 0
# echo "SELINUX=permissive" >/etc/selinux/config
```

## Installing Foreman

To install Foreman, you need to add a couple of RPM repositories,
install `foreman-installer`, and then run it.

More information here: https://theforeman.org/manuals/1.21/index.html#3.InstallingForeman

```
# yum install https://yum.puppetlabs.com/puppet5/puppet5-release-el-7.noarch.rpm
# yum install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# yum install https://yum.theforeman.org/releases/1.21/el7/x86_64/foreman-release.rpm

# yum update
# yum install foreman-installer

# foreman-installer --enable-foreman-plugin-remote-execution --enable-foreman-proxy-plugin-remote-execution-ssh

# firewall-cmd --add-service https
# firewall-cmd --add-service https --permanent
```

Write down the admin password that the installer outputs.

## Patching Foreman

The Cockpit integration into Foreman is done as part of the "Remote
Execution" plugin.  That plugin comes in two parts: one for Foreman
itself, one for the Smart Proxy.  We also need to update the Smart
Proxy itself.

So we need three patches, and some more gems to satisfy their
dependencies:

```
# yum install patch
# gem install net-ssh --version 4.2

# cd /opt/theforeman/tfm/root/usr/share/gems/gems/foreman_remote_execution-1.7.0/
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/master/rex.patch | patch -p1

# cd /usr/share/gems/gems/smart_proxy_remote_execution_ssh-0.2.0/
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/master/proxy-rex.patch | patch -p1

# cd /usr/share/foreman-proxy/
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/master/proxy.patch | patch -p1

# systemctl restart httpd
# systemctl restart foreman-proxy
```

## Setting up Cockpit

We will configure a special Cockpit instance just for Foreman, so we
need to install Cockpit.

```
# yum install cockpit
```

[ More precisely, we only need to install "cockpit-ws" on the machine
  where Foreman was installed, and on the machines that we want to
  access via Cockpit, we need to install whatever bits of Cockpit we
  actually want, but not "cockpit-ws".
]

This is the configuration for Cockpit itself:

```
# mkdir -p /etc/foreman-cockpit/cockpit/
# cat >/etc/foreman-cockpit/cockpit/cockpit.conf
[WebService]
LoginTitle = Foreman Cockpit
UrlRoot = /webcon/
Origins = https://foreman.demo.lan

[Bearer]
Action = remote-login-ssh

[SSH-Login]
command = /usr/libexec/foreman-cockpit-session

[OAuth]
Url = /cockpit/redirect
```

Note that the "Origins" value is the name of the machine.  If you have
used something else than "foreman.demo.lan", make sure to use the
right name there.

Also install the session helper:
```
# cd /usr/libexec/
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/master/foreman-cockpit-session >foreman-cockpit-session
# chmod a+x foreman-cockpit-session
```

This is for starting the special Cockpit:
```
# cat >/etc/systemd/system/foreman-cockpit.service
[Unit]
Description=Foreman Cockpit Web Service

[Service]
Environment=XDG_CONFIG_DIRS=/etc/foreman-cockpit/
ExecStart=/usr/libexec/cockpit-ws --no-tls --address 127.0.0.1 --port 9999
User=cockpit-ws
Group=cockpit-ws

[Install]
WantedBy=multi-user.target
```

[ It would be better to make this socket activated, of course, but we
  don't do that here for simplicity.
]

This instance of Cockpit is configured to be served by a reverse
proxy.  Thus, we need to enable `mod_proxy`.

```
# cat >/etc/httpd/conf.modules.d/proxy.load
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

```
# cat >/etc/httpd/conf.d/05-foreman-ssl.d/cockpit.conf
ProxyPreserveHost On
ProxyRequests Off

RewriteEngine On
RewriteCond %{HTTP:Upgrade} =websocket [NC]
RewriteRule /webcon/(.*)           ws://127.0.0.1:9999/webcon/$1 [P]
RewriteCond %{HTTP:Upgrade} !=websocket [NC]
RewriteRule /webcon/(.*)           http://127.0.0.1:9999/webcon/$1 [P]
```

Restart and enable as needed.

```
# systemctl enable --now foreman-cockpit
# systemctl restart httpd
```

## Configuring Foreman

Open `https://foreman.demo.lan` in a browser, accept the self-signed
certificate, and log in with the credentials given to you by the
installer.

If you don't have those credentials anymore, you can get new ones with
this command:

```
# foreman-rake permissions:reset
```

The installer has already entered the virtual machine and the smart
proxy into the database.  However, they might or might not be fully
associated with the right organization and location.  Make sure they
are both in the "Default Organization" and "Default Location".

[ Looks like a installer bug to me right now. More later on how to
  correct this, if needed.  For now, just poke around in the UI and
  try to figure it out. :-)
]

The Cockpit webserver will need to talk to the smart proxy, but this
doesn't work yet for https.  We need to configure the Smart Proxy and
its plugins to also allow connections with http.

```
# echo ":http_port: 8000" >>/etc/foreman-proxy/settings.yml
# echo ":trusted_hosts:" >>/etc/foreman-proxy/settings.yml
# sed -i -e 's/:enabled: https/:enabled: true/' /etc/foreman-proxy/settings.d/*
# systemctl restart foreman-proxy
```

Then edit the foreman.demo.lan smart proxy and change its URL to
"http://foreman.demo.lan:8000".  Click the "Refresh features" button
to see whether it still works.

Then go to "Administer / Settings / RemoteExecution" and change
"Cockpit URL" to `/webcon/=%{host}`.

[ Typing the "/" character into the text input field for the new value
  is impossible because the "/" yanks the focus to the filter text
  input.  But copy/paste works.
]

The last thing is to allow the remote execution plugin to log into the
virtual machine without password:

```
# ssh-copy-id -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy root@foreman.demo.lan
```

Now you can go to the page for the foreman.demo.lan host itself.  You
should see the "Web Console" button, and it should work.
