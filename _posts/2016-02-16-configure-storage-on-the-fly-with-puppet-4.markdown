---
layout: post
title:  "Configure Storage on the Fly with Puppet 4"
date:   2016-02-16 13:33:17 -0600
categories: puppet linux storage disk
---

Here's a snippet of puppet I'm using to configure storage. Now whenever I add new disks to the instance, puppet will expand the storage for me.

I apply this before I install the docker-engine package.

```ruby
package { 'lvm2': }

$::disks.each |$d, $v| {
  if ($d =~ /^sd[c-z]+/) {
    # Create pv if not a pv
    exec { "/sbin/pvcreate /dev/${d}":
      unless => "/sbin/pvs --noheadings /dev/${d}",
    }
    # Create VG if not exists
    exec { "/sbin/vgcreate ${vg} /dev/${d}":
      unless => "/sbin/vgs ${vg}",
    }
    # Add disk if not in the vg
    exec { "/sbin/vgextend ${vg} /dev/${d}":
      unless => "/sbin/pvs --noheadings -o vg_name /dev/${d} | /bin/grep ${vg}",
    }
  }
}

# create volume if it doesn't exist
exec { "/sbin/lvcreate --extents 100%FREE -n ${lv} ${vg}":
  unless  => "/sbin/lvs ${vg}/${lv}",
}

# Create ext4 filesystem
exec { "/sbin/mkfs.ext4 -j -b 4096 /dev/${vg}/${lv}":
  unless  => "/sbin/blkid /dev/${vg}/${lv} | /bin/grep 'TYPE=\"ext4\"'",
  require => Exec["/sbin/lvcreate --extents 100%FREE -n ${lv} ${vg}"],
}

# extend volume if room in data vg
exec { "/sbin/lvextend --extents +100%FREE ${vg}/${lv}":
  unless => "/sbin/vgs --noheadings -o vg_free ${vg} | /bin/grep -P '^\\s+0\\s
"
}

file { '/var/lib/docker':
  ensure =>; directory,
}

mount { '/var/lib/docker':
  ensure  => 'mounted',
  atboot  => true,
  device  => "/dev/${vg}/${lv}",
  fstype  => 'ext4',
  options => 'defaults,nobootwait,nobarrier',
  dump    => '0',
  pass    => '2',
  require => [
    File['/var/lib/docker'],
    Exec["/sbin/mkfs.ext4 -j -b 4096 /dev/${vg}/${lv}"],
  ]
}
```
