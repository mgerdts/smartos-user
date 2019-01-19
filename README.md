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
It creates a user named `joyent` with a home directory of `/zones/joyent`.  The
user gets the `Primary Administrator` profile, allowing anyone that logs in as
this user to execute arbitrary commands as root with `pfexec(1M)`.


```xml
 32     <!-- One instance per user -->
 33     <instance name="joyent" enabled="true">
 34       <!--
 35         This would be better with astring_list but those are only allowed in
 36         profiles, not manifests
 37       -->
 38       <property_group name="profiles" type="application">
 39         <propval name="admin" type="astring" value="Primary Administrator" />
 40       </property_group>
 41
 42       <property_group name="passwd" type="application">
 43         <propval name="dir" value="/zones/joyent" type="astring" />
 44         <propval name="comment" value="Joyent Engineering" type="astring" />
 45       </property_group>
 46     </instance>
```

You may also specify other `passwd(4)` fields such as `uid`, `gid`, and `shell`.
The password is not configurable with this.  That's OK, because `/etc/shadow` is
persisted across reboots.

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
