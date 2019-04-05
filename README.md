# Integrating Satellite and Cockpit

There is work in progress to integrate Cockpit into Satellite.
Eventually, setting this up should be trivial but until it's all
wrapped up and shipped -- it's complicated.

The following instructions will set up a single new virtual machine
that runs Satellite and you can seamlessly open Cockpit from Satellite
for that same virtual machine.

Current issues:

 - SELinux needs to be switched off.

## Setting up the virtual machine

Create a new RHEL 7.6 virtual machine with 8GiB RAM and 10GiB disk
from this, for example:

    https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.6/x86_64/product-software

In the installer, make sure that Network is switched on.  Also in the
installer, set the hostname of the new machine to "satellite.demo.lan".

Make sure that reverse DNS lookup gives consistent results inside the
virtual machine.  One easy way is to add a line like this to
`/etc/hosts` on the virtual machine itself:

```
192.168.100.150 satellite.demo.lan satellite
```

Make sure to use the real IP address of the virtual machine, of
course.

Make sure that the new virtual machine can also be reached via its
"satllite.demo.lan" name from the outside.  Again, a easy way is to
add a line like above to `/etc/hosts` on the machine that will run the
browser.

Switch off SELinux permanently:

```
# setenforce 0
# echo "SELINUX=permissive" >/etc/selinux/config
```

## Installing Satellite

To install Satellite, you need to subscribe to a couple of RPM repositories,
install `satellite`, and then run its installer.

```
# subscription-manager register
# subscription-manager list --available --matches 'Red Hat Satellite 6 Beta'
```

Pick one of the pools and attach to it.

```
# subscription-manager attach --pool=<pool_id>
# subscription-manager repos --disable "*"
# subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-server-rhscl-7-rpms --enable=rhel-server-7-satellite-6-beta-rpms --enable=rhel-7-server-satellite-maintenance-6-beta-rpms --enable=rhel-7-server-ansible-2.6-rpms
# subscription-manager release --unset

# yum update
# yum install satellite

# satellite-installer --scenario satellite --enable-foreman-plugin-remote-execution --enable-foreman-proxy-plugin-remote-execution-ssh

# firewall-cmd --add-service https
# firewall-cmd --add-service https --permanent
```

Write down the admin password that the installer outputs.

## Patching Foreman

The Cockpit integration into Satellite is done as part of the "Remote
Execution" plugin of Foreman.  That plugin comes in two parts: one for
Foreman itself, and one for the Smart Proxy ("Capsule" in Satellingo,
but we will keep calling it a proxy here).

So we need two patches, and one more gems to satisfy their
dependencies:

```
# gem install net-ssh --version 4.2

# cd /opt/theforeman/tfm/root/usr/share/gems/gems/foreman_remote_execution-1.6.7/
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/satellite/rex.patch | patch -p1

# cd /usr/share/gems/gems/smart_proxy_remote_execution_ssh-0.2.0/
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/satellite/proxy-rex.patch | patch -p1

# systemctl restart httpd
# systemctl restart foreman-proxy
```

## Setting up Cockpit

We will configure a special Cockpit instance just for Satellite, so we
need to install Cockpit.

```
# yum install cockpit
```

[ More precisely, we only need to install "cockpit-ws" on the machine
  where Satellite was installed, and on the machines that we want to
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
Origins = https://satellite.demo.lan

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
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/satellite/foreman-cockpit-session >foreman-cockpit-session
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
proxy.  We also need to enable `mod_proxy_wstunnel`.

```
# cat >/etc/httpd/conf.modules.d/proxy_wstunnel.load
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
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

## Configuring Satellite

Open `https://satellite.demo.lan` in a browser, accept the self-signed
certificate, and log in with the credentials given to you by the
installer.

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
# ssh-copy-id -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy root@satellite.demo.lan
```

Now you can go to the Satellite page for the satellite.demo.lan host
itself.  You should see the "Web Console" button, and it should work.
