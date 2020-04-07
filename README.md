# Foreman and Cockpit

There used to be work in progress to integrate Cockpit into Foreman,
and we would record the necessary manual steps here to make the it all
work.  Now it's all wrapped up and shipped, and things are a lot simpler.

Still, it's good to document exactly what is supposed to work and how.

The following instructions will set up a single new virtual machine
that runs Foreman and you can seamlessly open Cockpit from Foreman for
that same virtual machine.

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

## Installing Foreman

To install Foreman, you need to add a couple of RPM repositories,
install `foreman-installer`, and then run it.

More information here: https://theforeman.org/manuals/1.24/quickstart_guide.html

```
# yum install https://yum.puppet.com/puppet6-release-el-7.noarch.rpm
# yum install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# yum install https://yum.theforeman.org/releases/1.24/el7/x86_64/foreman-release.rpm

# yum update
# yum install foreman-installer

# foreman-installer --enable-foreman-plugin-remote-execution \
                    --enable-foreman-proxy-plugin-remote-execution-ssh \
                    --enable-foreman-plugin-remote-execution-cockpit

# firewall-cmd --add-service https
# firewall-cmd --add-service https --permanent
```

Write down the admin password that the installer outputs.

## Setting up password-less remote execution

In order to seamlessly open Cockpit on a host, you need to be able to
execute remote commands on it without providing any credentials.

You can store the credentials in the `remote_execution_ssh_user` and
`remote_execution_ssh_password` host parameters and Foreman will use
these to connect to Cockpit.

Instead of storing the password in a host parameter, you can also copy
the public key of the proxy over to the host:

```
# ssh-copy-id -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy root@foreman.demo.lan
```

Now you can go to the page for the foreman.demo.lan host itself.  You
should see the "Web Console" button, and it should work.

## Addendum: Puma

In the future, the Foreman smart proxy will offer a choice between
Puma and Webrick.  The Cockpit integration needs to be adapted for
Puma, and here is how you can test that.

Patch the smart proxy to support Puma:

```
# yum install patch
# cd /usr/share/foreman-proxy
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/master/proxy-puma.patch | patch -p1
```

Install the Puma gem:

```
# yum install gcc ruby-devel openssl-devel
# gem install puma -v 3.10.0
```

Tell the proxy to use Puma:

```
# echo >>/etc/foreman-proxy/settings.yml ":http_server_type: 'puma'"
```

Tell the proxy to only listen on IPv4, because IPv6 is nothing but trouble:

```
# sed -i /etc/foreman-proxy/settings.yml -e 's/:bind_host: .*/:bind_host: "0.0.0.0"/'
```

Finally, patch the REX plugin to support SSH sessions also with Puma and restart:

```
# cd /usr/share/gems/gems/smart_proxy_remote_execution_ssh-0.2.1/
# curl https://raw.githubusercontent.com/mvollmer/foreman-cockpit/master/proxy-rex-puma.patch | patch -p1
# systemctl restart foreman-proxy
```
