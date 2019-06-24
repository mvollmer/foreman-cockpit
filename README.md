# Integrating Foreman and Cockpit

There is work in progress to integrate Cockpit into Foreman.
Eventually, setting this up should be trivial but until it's all
wrapped up and shipped -- it's complicated.

Foreman 1.22 is the first version that includes the necessary code for
the Cockpit integration, but the foreman-installer can not yet create
the necessary configuration files.

The following instructions will set up a single new virtual machine
that runs Foreman and you can seamlessly open Cockpit from Foreman for
that same virtual machine.


Current issues:

 - SELinux needs to be switched off.

## Setting up the virtual machine

Create a new Centos 7 virtual machine with 4GiB RAM and 10GiB disk
from this:

    http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso

In the installer, make sure that Network is switched on.  Also in the
installer, set the hostname of the new machine to "foreman.demo.lan".

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

More information here: https://theforeman.org/manuals/1.22/quickstart_guide.html

```
# yum install https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
# yum install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# yum install https://yum.theforeman.org/releases/1.22/el7/x86_64/foreman-release.rpm

# yum update
# yum install foreman-installer

# foreman-installer --enable-foreman-plugin-remote-execution --enable-foreman-proxy-plugin-remote-execution-ssh

# firewall-cmd --add-service https
# firewall-cmd --add-service https --permanent
```

Write down the admin password that the installer outputs.

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
command = /opt/theforeman/tfm/root/usr/share/gems/gems/foreman_remote_execution-1.8.0/extra/cockpit/foreman-cockpit-session

[OAuth]
Url = /cockpit/redirect
```

Note that the "Origins" value is the name of the machine.  If you have
used something else than "foreman.demo.lan", make sure to use the
right name there.

Also configure the session helper:

```
# cat >/etc/foreman-cockpit/settings.yml
:foreman_url: https://foreman.demo.lan

:ssl_ca_file: /etc/puppetlabs/puppet/ssl/certs/ca.pem
:ssl_certificate: /etc/puppetlabs/puppet/ssl/certs/foreman.demo.lan.pem
:ssl_private_key: /etc/puppetlabs/puppet/ssl/private_keys/foreman.demo.lan.pem
```

This is for starting the special Cockpit:
```
# cat >/etc/systemd/system/foreman-cockpit.service
[Unit]
Description=Foreman Cockpit Web Service

[Service]
Environment=XDG_CONFIG_DIRS=/etc/foreman-cockpit/
ExecStart=/usr/libexec/cockpit-ws --no-tls --address 127.0.0.1 --port 9999
User=foreman
Group=foreman

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
# systemctl restart httpd
# systemctl enable --now foreman-cockpit
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

Go to "Administer / Settings / RemoteExecution" and change "Cockpit
URL" to `/webcon/=%{host}`.

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
