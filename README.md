# Introduction

Standalone metadata server to simplify the use of vendor cloud
images with a standalone kvm/libvirt server

- supports a subset of the EC2 metadata "standard" (as
  documented by Amazon), compatible with the EC2 data source
  as implemented by cloud-init
- allows the user to configure cloud-init via user-data
- cloud-init config can be templated, making use of data from
  the system configuration as well as information about the host
  itself
- provides IP address and DNS management via dnsmasq, including
  generating hostnames from instance names and supporting DNS
  resolution of the generated names
- integrates with libvirt to automatically update records as
  new instances come online

See the sample config file for the full set of configuration
options.

# Dependencies

mdserver as of 0.6.0 only supports Python3 - if you need to run with
Python2 use version 0.5.1 or later.

Package dependencies:

- bottle (>= 0.12.0)
- xmltodict (>= 0.9.0)

# Quick Start

Decide on a network that you'll use for the metadata network - the
default is 10.122.0.0/16, which will generally work fine.

Create a bridge which will host this network - the default is
virbr0:
```
# brctl addbr virbr0
```
You will need to add an interface connected to this bridge to all
instances you want to manage using mdserver.

Add the default gateway and EC2 "magic" IP addresses to the bridge:

```
# ip addr add 10.122.0.1/16 dev virbr0
# ip addr add 169.254.169.254 dev virbr0
```

To install requirements using pip run the following:

```
# pip install -r requirements.txt
```

Depending on your target system, you can either install directly
or by building an RPM package and installing that.

To install directly from the source:

```
# python setup.py install
```

To build an RPM package and install the package:

```
# python setup.py bdist_rpm
# rpm -ivh dist/mdserver-<version>.noarch.rpm
```

The configuration file will be installed by default in
`/etc/mdserver/mdserver.conf`, along with a daemon config file in
`/etd/default/mdserver`. The default configuration will not be
very useful - you will need to at last configure in the correct
network details, as well as setting up userdata templates for your
instances. You will also want to add your root/admin user's ssh
key as default, if you plan to use ssh to log into your
instances.

User data files are sourced by default from
`/etc/mdserver/userdata`.

mdserver assumes that it's running in a systemd context, though it
doesn't strictly rely on any systemd features - in particular, it
uses systemd's support for defining relationships between units in
order to manage dnsmasq. In a non-systemd context this can be
emulated within a traditional init script, but this is not an
explicitly supported use case.

The supplied systemd unit files should work if the configuration is
only lightly edited - changing the base dir will require adjustements
to the unit files.

Starting the system can be done in the expected way:
```
# systemctl start mdserver
```
This will bring up both mdserver and dnsmasq

The server can also be run manually:

```
/usr/local/bin/mdserver /etc/mdserver/mdserver.conf
```

In this case you will need to also start dnsmasq:

```
/usr/sbin/dnsmasq --conf-file /var/lib/mdserver/dnsmasq/mds.conf
```

In all cases you will need to ensure that the libvirt hook script
is installed in the appropriate location - typically this is
`/etc/libvirt/hooks/qemu` - and it will need to be made executable.

The libvirt hook relies on the `/etc/default/mdserver` file to know
how to communicate with the mdserver process - however you configure
the mdserver to listen, it needs to match the address in the default
file.

Finally, by default logs go to `/var/log/mdserver.log`.

# Enabling cloud-init

Vendor supplied cloud images using newer versions of cloud-init
will not recognise md_server as a valid metadata source at the
moment, and will thus not even attempt to configure the instance.
This can be worked around in two ways: either make your instance
look like an AWS instance, by setting appropriate BIOS data, or
by editing the image to force cloud-init to run after the network
is up.

## Pretending to be AWS
Cloud-init determines that it's running on an AWS instance by
looking at the BIOS serial number and uuid values: they must be
the same string, and the string must start with 'ec2'. This can
be achieved by adding something like the following snippet to
your domain XML file:

```
<os>
  ...other os data...
  <smbios mode='sysinfo'/>
</os>
<sysinfo type='smbios'>
  <system>
    <entry name='manufacturer'>Plain Old Virtual Machine</entry>
    <entry name='product'>Plain old VM</entry>
    <entry name='serial'>ec242E85-6EAB-43A9-8B73-AE498ED416A8</entry>
    <entry name='uuid'>ec242E85-6EAB-43A9-8B73-AE498ED416A8</entry>
  </system>
</sysinfo>
```

The uuid must be valid, so the easiest way to create this string
is to generate a fresh uuid and replace the first three characters
with 'ec2'.

## Forcing cloud-init to run

Cloud-init can be forced to run by editing the systemd configuration
in the instance. This can be achieved by running the following
commands in the image (probably using something like guestfish):

```
# cat <<EOF >/etc/systemd/network/default.network
[Match]
Type=en*
Name=ens3

[Network]
DHCP=yes
EOF
# ln -s /lib/systemd/system/cloud-init.target /etc/systemd/system/multi-user.target.wants/
```

The network interface named in `default.network` must be on the
mds network for this to work.

# Usage

## Initialising the Database

mdserver maintains a persistent database, typically stored in
`/var/lib/mdserver/`, from which it gets the information that it
needs to respond to requests. A clean install of mdserver will
have an empty database, which must be initialised before mdserver
can respond usefully to anything.

Initialising the database is done by uploading the full domain
XML for each instance that wants to use it. The domain XML for an
instance can be acquired using the following command:
```
virsh dumpxml instance1 > instance1.xml
```
The resulting XML file can be uploaded to the mdserver using a
simple curl command:
```
curl -s -d @instance1.xml http://169.254.169.254/instance-upload
```
The mdserver will parse the XML file, extract the information it
needs, allocate an IP address, and then store that information in
its database. It will then update the dnsmasq DHCP and DNS files so
that when the instance comes up and attempts to get on the network
it will receive a known IP address from dnsmasq, and its hostname
will resolve to that IP address in a DNS lookup.

Thanks to the libvirt hook script any new instances will be uploaded
at start up, so this is a one time task (though this process can be
used to update the database if so desired).

## Request Handling

When cloud-init runs on boot it will attempt to contact an EC2
metadata server on the "magic" IP 169.254.169.254:80. mdserver
listens on this address for requests and generates a response based
on information from its database, using the source IP address of the
request to locate the host data in the database.

Most of the requests are quite simple, responding with a single line
generated from the database. However, the user-data request is far
more involved.

When mdserver receives a user-data request it starts by resolving
the instance in the database, and then searches for a file in the
userdata directory (typically `/etc/mdserver/userdata/`) using
the following filenames:

- `<userdata_dir>/<instance>`
- `<userdata_dir>/<instance>.yaml`
- `<userdata_dir>/<MAC>`
- `<userdata_dir>/<MAC>.yaml`

A default template userdata file can be also specified in the
configuration which will be used as a fallback if nothing more
specific is found - this is typically something like
`<userdata_dir>/base.yaml`. If the default template path is not set
then a minimal hard-coded template will be used instead.

Once the template to use is determined it is processed using
Bottle's Simple Template library, with details about the instance
made available to the template processor along with the following
values from the mdserver configuration:

- all public keys, in the form `public_key_<entry name>`
  i.e. an entry in the `[public-keys]` section named `default` will
  be available in the userdata template as a value named
  `public_key_default`
- a default password (`mdserver_password`) - only if set by the
  user!
- the hostname (`hostname`)

Additional key-value data to be made available to the template
can be specified in the `[template-data]` section of the config
file. e.g:

```
[template-data]
foo=bar
```

would result in `bar` being added to the template data under the
key `foo`.

Values can be interpolated into the file using the `{{<key>}}`
syntax - more sophisticated template behaviour can be used, see
the Bottle templating engine documentation for more details.

The output of the template processing is then returned to the
client.

# DNS Management

By default mdserver will generate a DNS hosts file that dnsmasq will
read and track updates to over time. This means that if you attempt
to resove the name of an instance through this dnsmasq instance you
will get the correct IP address, and vice-versa for A lookups.

By default the dnsmasq DHCP configuration does not specify any DNS
servers, but it can be configured to specify the mdserver-managed
dnsmasq instance as a DNS server by setting adding
```
[dnsmasq]
use_dns=yes
```
to the mdserver configuration. Since dnsmasq acts as a forwarding
resolver this will generally work without issues, however the
reliability in any given network cannot be guaranteed.

Adding the dnsmasq instance to the hypervisor resolv.conf should also
work without issues, but again the exact details of performance and
reliability will depend on the local circumstances.
