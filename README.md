# smartos-user SMF service

This repository contains an SMF service that may be used to make a non-standard
user persist across reboots of a SmartOS compute node.

## Setup

```
cn# mkdir -p /opt/custom/{bin,smf}
```

Copy the contents of the `bin` and `smf` directories from this repostority into
the directories created above.

You may customize the set of users and created and various attributes of them by
modifying the service manifest.

```
cn# vi /opt/custom/smf/smartos-user.xml
```

This instance is in the manifest - you probably want to customize or replace it.
It creates a user named `user1` with a home directory of `/zones/user1`.  The
user gets the `Primary Administrator` profile, allowing anyone that logs in as
this user to execute arbitrary commands as root with `pfexec(1M)`.


```xml
    <!-- One instance per user -->
    <instance name="user1" enabled="true">
      <!--
        If the user should have any profiles assigned, include the 'profiles'
        property group.  In the example above, this user will have the 'Primary
        Administrator' profile, which will allow root access.  See profiles(1).
      -->
      <property_group name="profiles" type="application">
        <propval name="admin" type="astring" value="Primary Administrator" />
      </property_group>

      <property_group name="passwd" type="application">
        <propval name="dir" value="/zones/user1" type="astring" />
        <propval name="shell" value="/bin/bash" type="astring" />
        <!-- Other properties you may wish to configure. See passwd(4). -->
        <!-- <propval name="uid" value="123" type="integer" /> -->
        <!-- <propval name="gid" value="123" type="integer" /> -->
        <!-- <propval name="comment" value="First User" type="astring" /> -->
      </property_group>

      <property_group name="shadow" type="application">
        <!--
          Uncomment the following to set the password to "8SPPYvsTGxytE7s6".
          Warning! Whatever you put here is world readable.  Use of ssh keys
          instead is advised.
        -->
        <!-- <propval name="pass" type="astring"
          value="$5$ASERtDf8$PT5CgDKf9Vk4mJAZsgAeC.j.E21XHL6Fkprkle3uIAA" /> -->
      </property_group>
    </instance>
```

**WARNING:** If there are multiple instances in the service, the start method
script (`/opt/custom/bin/add-smartos-user`) is not hardened against races.  It
is suggested that each instance have a dependency on the previous one so that
they are executed sequentially.  For instance, if I were to add a personal home
directory after the the one shown above, it may look like this:

```xml
    <instance name="mgerdts" enabled="true">
      <dependency name="joyent" type="service" restart_on="none" grouping="require_all">
        <service_fmri value="svc:/smartdc/smartos-user:joyent" />
      </dependency>
      <property_group name="passwd" type="application">
        <propval name="dir" value="/zones/mgerdts" type="astring" />
        <propval name="shell" value="/bin/bash" type="astring" />
        <propval name="comment" value="Mike Gerdts" type="astring" />
      </property_group>
    </instance>
```

You can validate that the changes to the manifest are well-formed with `svccfg
validate`.  If it gives no output, all is well.

```
[root@hostname ~]# svccfg validate /opt/custom/smf/smartos-user.xml
```

Once the new service manifest validates, it may be imported.

```
[root@hostname ~]# svccfg import /opt/custom/smf/smartos-user.xml
```

You can then verify that it is doing its work with `svcs`.

```
[root@hostname ~]# svcs smartos-user
STATE          STIME    FMRI
offline*       21:31:04 svc:/smartdc/smartos-user:user1
[root@hostname ~]# svcs smartos-user
STATE          STIME    FMRI
online         21:31:08 svc:/smartdc/smartos-user:user1
```

Once it is online, you can see that the user was created.  From `shadow(4)` we
can see that this account has no password by the `NP` in the second field.  This
means that the user can only log in via ssh key-based authentication.

```
[root@hostname ~]# getent passwd user1
user1:x:500:10::/zones/user1:/bin/bash
[root@hostname ~]# grep ^user1: /etc/shadow
user1:NP:17919::::::
```

To set up key-based authentication, place the appropriate public key the user's
`.ssh/authorized_keys` file.

```
[root@hostname ~]# su - user1
- SmartOS (build: 20190118T192019Z)
-bash-4.3$ mkdir -m 700 .ssh
-bash-4.3$ vi .ssh/authorized_keys
-bash-4.3$ chmod 600 .ssh/authorized_keys
```

## Example: User to kick of a script

Suppose a CI/CD pipeline needs to trigger an operation as root on a headnode.
This example creates a user named kicker that kicks off the job using command
line options that are provided by the CI/CD pipeline.  It is not possible for
the user to perform any other actions.

The content of `/opt/custom/smf/smartos-user.xml` is

```xml
<?xml version="1.0"?>
<!DOCTYPE service_bundle SYSTEM "/usr/share/lib/xml/dtd/service_bundle.dtd.1">

<!--
  This Source Code Form is subject to the terms of the Mozilla Public
  License, v. 2.0. If a copy of the MPL was not distributed with this
  file, You can obtain one at http://mozilla.org/MPL/2.0/.

  Copyright (c) 2019 Joyent, Inc.

  This service creates a user per service instance on each boot.  In order for
  the user to be useful, the password must be set or ssh keys must be present.

  When using this as standalone SmartOS (not as a Triton Compute Node),
  /etc/shadow is preserved across reboots. As such, with SmartOS it is possible
  to preserve the password across reboots while keeping its hash secret.  As a
  Triton Compute Node, you must either specify the shadow/pass property in each
  instance or maintain per-user .ssh/authorized_keys.
-->

<service_bundle type="manifest" name="smartos-user">
  <service name="smartdc/smartos-user" type="service" version="1">
    <exec_method type="method" name="start"
      exec="/opt/custom/bin/add-smartos-user %i" timeout_seconds="30" />
    <exec_method type="method" name="stop" exec=":true" timeout_seconds="60" />

    <!--
      Do not worry about dependencies.  Since this lives in zones/opt, there's
      no chance that it can be run too early.
    -->
    <property_group name="startd" type="framework">
      <propval name="duration" type="astring" value="transient" />
      <propval name="ignore_error" type="astring" value="core,signal" />
    </property_group>

    <!-- One instance per user -->
    <instance name="kicker" enabled="true">
      <property_group name="profiles" type="application">
        <propval name="admin" type="astring" value="Primary Administrator" />
      </property_group>

      <property_group name="passwd" type="application">
        <propval name="dir" value="/zones/kicker" type="astring" />
        <propval name="shell" value="/bin/bash" type="astring" />
        <propval name="comment" value="CI/CD Job Runner" type="astring" />
      </property_group>
    </instance>

    <stability value="Evolving" />
    <template>
      <common_name>
        <loctext xml:lang="C">Joyent User Creator</loctext>
      </common_name>
    </template>
  </service>
</service_bundle>
```

Validate then import the manifest.  Ensure the service is online.

```
[root@hostname ~]# svccfg validate /opt/custom/smf/smartos-user.xml
[root@hostname ~]# svccfg import /opt/custom/smf/smartos-user.xml
[root@hostname ~]# svcs smartos-user
STATE          STIME    FMRI
online         21:49:26 svc:/smartdc/smartos-user:kicker
```

Put the command that will be run into the right place.

```
[root@hostname ~]# su - kicker
- SmartOS (build: 20190118T192019Z)
-bash-4.3$ mkdir bin
-bash-4.3$ vi bin/cicd-thingy
-bash-4.3$ chmod 755 bin/cicd-thingy
-bash-4.3$ cat bin/cicd-thingy
#! /bin/ksh

export PATH=/usr/bin:/usr/sbin

# It is not possible to pass args, but they can be read from stdin
while read thing1 thing2; do
        echo "Got thing1=$thing1 thing2=$thing2"
done
```

The script above is really simple. If you need to run a command as root, prefix
it with `pfexec`.

Create ssh key pair elsewhere.

```
cicd@cicdhost$ ssh-keygen -P "" -f .ssh/cicd-runner
Generating public/private rsa key pair.
Your identification has been saved in .ssh/cicd-runner.
Your public key has been saved in .ssh/cicd-runner.pub.
The key fingerprint is:
SHA256:yfDACrWyBJ/lbq0oqdY03Jqidk9RxegKEZxesaflGVw cicd@cicdhost
The key's randomart image is:
+---[RSA 2048]----+
|. ..=.. oE       |
| o O +o.o.       |
|  B =.=*         |
| . B o**o.       |
|  o *oooS        |
| . * +.          |
|o + =.           |
|.= =.            |
|* o ..           |
+----[SHA256]-----+
$ cat .ssh/cicd-runner.pub
ssh-rsa AAAAB3NzaC1yc2EAAA...  cicd@cicdhost
```

Add the public key to `.ssh/authorized_keys`, including a `command`.  See
`AUTHORIZED KEYS FILE FORMAT` in `sshd(1M)` for details.  This is runniung as
the newly created kicker user.

```
-bash-4.3$ mkdir -m 700 .ssh
-bash-4.3$ vi .ssh/authorized_keys
-bash-4.3$ chmod 400 .ssh/authorized_keys
-bash-4.3$ cat .ssh/authorized_keys
command="/zones/kicker/bin/cicd-thingy" ssh-rsa AAAAB3NzaC1yc2EAAA...  cicd@cicdhost
```

Now we are ready to test it.  As the cicd user:

```
cicd@cicdhost$ echo 'a\ b stuff' | ssh -l kicker -i .ssh/cicd-runner smartos-host
Got thing1=a b thing2=stuff
```
