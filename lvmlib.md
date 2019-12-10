### LVM library discussion

#### Why LVM needs a library
 * Economy  of scale, with virtualization/cloud computing systems are provisioned  and managed by automated tools/frameworks.  Other applications have  become a big consumer.  Tools must be automatable.
 * Each invocation of the command line causes a new process and subshell.
 * It's a pain to do string parsing/manipulation in other C programs.
 * System locale settings change output causing code which parses stdout/strerr to fail.
   * https://bugs.launchpad.net/cinder/+bug/1219963
 * LVM command line output/formatting changes break applications.
   * https://bugs.launchpad.net/cinder/+bug/1470218
 * Reporting errors is limited to exit values and stderr messages which many users simply ignore.  
   * Actual code ( openstack nova[1] ): 
   * cmd = ('lvcreate', '-L', '%db' % size, '-n', lv, vg) 
   * execute(*cmd, run_as_root=True, attempts=3)
   * OpenStack/Cinder has numerious error paths which retry operations.  Some error paths call into udev settle to work around suspected race conditions with udev handling
   * If all you offer is an exit code of 0 or 1, it forces a user to check every possible precondition, some of which the user may not possibly know, thus they may be forced to look at stderr for error context
 * Command line wrappers either have to pass configuration options on the command line or use a separate config file to insulate themselves from changes to main lvm.conf file.
 * Calling  lvm command line tools in other programs results in hidden package  dependencies which are not expressed in the package manager. This  results in broken application installation.
 * Command line tools are required to exist even on embedded platforms/containers, limits usability.
 * Security, no command line application results in no command line injection opportunities and less security surface.
 * Reduction  in duplication of work.  SSM, Blivet, OpenStack (cider & nova),  Xen, Cockpit, oVirt, Gluster and many others write their own wrappers to  make an API for the LVM command line.
 * Users are electing to use the device mapper library directly instead of utilizing LVM.

#### Requested features/requirements, (feedback from actual LVM users)
 * Library needs feature parity with command line.  The library needs to be an integral part that the command line uses.
 * C  & python bindings, C being more important.  GObject introspection  would be good as other language support could be supported for 'free'.
 * Would be ideal if API could abstract or hide LVM implementation details.
 * Functions  that are easy to use for most common use cases in addition to functions  that allow the user to exploit all functionality
 * The  library needs to have the notion of job control.  The ability to start a  long running operation eg. pvmove and be able to monitor its status,  percent complete and cancel it if needed with the ability to report  errors.
 * Retrieving the state of the LVM should work well with large installations, under load, and for clients that are started on-demand (thus, with low  initialization overhead).
 * A reliable way to get change notifications, udev is lacking.
 * Safe for use for multi-threaded apps.
 * Ability  to calculate or predict how much space is needed for metadata.  Would  be better if user didn't need to concern them with this at all.
 * Information about thread-safety and locking with the command line tools.
 * A small embeddable C sub library which provides read-only meta data for other projects to utilize (eg. GRUB).

#### Design/implementation considerations for a C library
##### (Taken from libabc & others)
 * Zero global state, thread aware, but not thread safe
 * Never export global variables
 * Use a library context object
 * No static variables defined in functions
 * Use common unique prefix to prevent name space collisions
 * Not expose complex data structures or return C structs from functions
 * Use opaque data structures
 * Use common standard function names for typical operations
 * No function callbacks
 * Never call exit, abort or assert
 * Never output to stdout/stderr
 * Use symbol versioning and hide all internal symbols
 * Never fork()/exec()
 * Make your code safe for unexpected termination and any point, don't leave temp files around etc.
 * Executing out-of-process tools and parsing their output is not acceptable in a library
 * Don't expose fixed length strings
 * Stable, devoid of memory leaks, memory corruption etc., client application shares the same process space

#### Examples of other APIs
In creating a library interface for lvm, it may be useful to look at
what others have done with the lvm command line, to see what abstraction
they have done.  So the usual suspects would include:

 * git://git.fedorahosted.org/blivet.git
 * git://github.com/openstack/cinder.git
 * git://git.code.sf.net/p/storagemanager/code
 * https://github.com/cockpit-project/storaged.git
 * git://libvirt.org/libvirt.git

#### Other examples of storage libraries include:

 * Oracle® ZFS Storage Appliance RESTful API:http://docs.oracle.com/cd/E51475_01/html/E52433/index.html
 * An example file from libzfs, the local zfs API:https://hg.openindiana.org/upstream/illumos/illumos-gate/file/tip/usr/src/lib/libzfs/common/libzfs_pool.c
 * Windows storage mgmt APIhttp://msdn.microsoft.com/en-us/library/windows/desktop/hh830613%28v=vs.85%29.aspx

#### Who is using lvm and what specific operations are they doing with it
 * Anaconda/blivet
   * pvcreate --dataalignment 1024k <device>
   * pvresize --setphysicalvolumesize <size>m
   * pvremove --force --force --yes <device>
   * pvscan --cache <device>
   * pvmove <src> <dest>
   * pvs --unit=k --nosuffix --nameprefixes --unquoted --noheadings -o pv_name,pv_uuid,pe_start,vg_name,vg_uuid,vg_size,vg_free, vg_extent_size,vg_extent_count,vg_free_count,pv_count
   * vgcreate -s <size>m
   * vgremove --force <vg_name>
   * vgchange -a y <vg_name>
   * vgchange -a n <vg_name>
   * vgreduce --removemissing --force <vg_name>
   * vgextend <vg_name> <pv>
   * vgs --noheadings --nosuffix --nameprefixes --unquoted --units m -o uuid,size,free,extent_size,extent_count,free_count,pv_count
   * lvs -a --unit k --nosuffix --nameprefixes --unquoted --noheadings -o vg_name,lv_name,lv_uuid,lv_size,lv_attr,segtype
   * lvs --noheadings -o origin <vg_name>/<lv_name>
   * lvcreate -L <size>m -n <lv_name> -y <vg_name> <pvs>
   * lvremove --force --yes <vg_name>/<lv_name>
   * lvresize --force -L <size>m <vg_name>/<lv_name>
   * lvchange -a y <vg_name>/<lv_name>
   * lvchange -a n <vg_name>/<lv_name>
   * lvconvert --merge <vg_name>/<lv_name>
   * lvcreate -s -L <size>m -n <snap_name> <vg_name>/<lv_name>
   * lvcreate --thinpool <vg_name>/<lv_name> --size <size>m --poolmetadatasize <size in mb> --chunksize <size in kb> --profile <profile>
   * lvcreate --thinpool <vg_name>/<pool_name> --virtualsize <size>m -n <lv_name>
   * lvcreate -s -n <snap_name> <vg_name>/<origin_name> --thinpool <pool name>
   * lvs --noheadings -o pool_lv <vg_name>/<lv_name>
 * Storaged/cockpit (Uses both the lvm2app library and command line)
   * lvm2app library function calls in use (see src/helper.c)
     * lvm_init
     * lvm_quit
     * lvm_list_vg_names
     * lvm_lv_get_name
     * lvm_lv_get_uuid
     * lvm_lv_get_size
     * lvm_pv_get_name
     * lvm_pv_get_uuid
     * lvm_pv_get_size
     * lvm_pv_get_free
     * lvm_vg_get_name
     * lvm_vg_get_uuid
     * lvm_vg_get_size
     * lvm_vg_get_free_size
     * lvm_vg_get_extent_size
     * lvm_vg_list_pvs
     * lvm_vg_list_lvs
     * lvm_lv_get_property
       * Specific properties passed to lvm_lv_get_property
         * lv_attr
         * lv_path
         * move_pv
         * pool_lv
         * origin
         * data_percent
         * metadata_percent
         * copy_percent
     * lvm_vg_open
     * lvm_vg_close
   * command line (searched for calls to storage_daemon_launch_spawned_job)
     * lvremove -f <lv name>
     * lvrename <prev> <new>
     * lvresize <vg>/<lv> -L<size>b -r
     * lvchange <vg>/<lv> -ay -K --yes
     * lvchange <vg>/<lv> -an -K --yes
     * lvcreate -s <vg>/<lv> -n <name> -L<size>b
     * vgrename <prev> <new>
     * vgextend <vg> <pv>
     * pvmove <pv>
     * lvcreate <vg> -T -L <size>b --thinpool <name>
     * lvcreate <vg> --thinpool <lv name> -V <size>b -n <name>
 * Cinder (cinder/brick/local_dev/lvm.py)
   * vgs --noheadings -o name <vg_name>
   * vgcreate <vg_name> <pv(s)>
   * vgs --noheadings -o uuid <vg_name>
   * lvs --noheadings --unit=g -o size,data_percent --separator : --nosuffix /dev/<vg name>/<thin pool name>
   * vgs --version
   * lvs --noheadings --unit=g -o vg_name,name,size --nosuffix <optional vg name>
     * (Note: they log a warning if operation takes > 60 seconds
   * pvs --noheadings --unit=g -o vg_name,name,size,free --separator : --nosuffix
   * vgs --noheadings --unit=g -o name,size,free,lv_count,uuid --separator : --nosuffix <optional vg name>
     * (Note: they log a warning if operation takes > 60 seconds
   * lvcreate -T -L <size> <vg pool name>
   * lvcreate -T -V <size> -n <name> <pool>
     * optionally add: -m <count> --nosync --mirrorlog mirrored -R <size if size > 1.5T>
   * lvcreate -n name <vg_name> -L <size>
     * optionally add: -m <count> --nosync --mirrorlog mirrored -R <size if size greater than 1.5T >
   * lvcreate --name <name> --snapshot <vg>/<lv>
   * lvcreate --name <name> --snapshot <vg/<lv> -L <size>g
   * lvchange -a y --yes
   * lvchange -a y --yes -K
   * lvremove --config 'activation { retry_deactivation = 1} ' -f <vg>/<lv>
   * lvremove --config 'activation { retry_deactivation = 1} devices { ignore_suspended_devices = 1} ' -f <vg>/<lv>
   * lvconvert --merge <snapshot name>
   * lvdisplay --noheading -C -o Attr <vg>/<name>  #Checking to see if lv has snapshot
   * lvextend -L <new size> <vg>/<lv>
   * lvrename <vg name> <lv name> <lv new name>
 * Docker (dev mapper function calls, maybe Mike or others can supply high level needs here)
   * dm_log_with_errno_init
   * dm_task_destroy
   * dm_task_create
   * dm_task_run
   * dm_task_set_name
   * dm_task_set_message
   * dm_task_set_sector
   * dm_task_set_cookie
   * dm_task_set_add_node
   * dm_task_set_ro
   * dm_task_add_target
   * dm_task_get_deps
   * dm_task_get_info
   * dm_task_get_driver_version
   * dm_get_next_target
   * dm_udev_wait
   * dm_log_init_verbose
   * dm_set_dev_dir
   * dm_get_library_version
 * Snapper (C++ code using command line)
   * lvcreate --permission -r snapshot --name <ss name> <vg>/<lv>
   * lvremove --force <vg>/<lv>
   * lvs --noheadings -o lv_attr,segtype <vg>/<lv>
   * lvs --noheadings -o lv_name,lv_attr,segtype <vg>/<lv>
   * lvs --noheadings -o lv_name,lv_attr,segtype <vg>
   * lvchange -ay <vg>/<lv>
   * lvchange -an <vg>/<lv>
 * VDSM/oVirt
   * pvs --noheadings --units b --nosuffix --separator | --ignoreskippedcluster -o uuid,name,size,vg_name,vg_uuid,pe_start,pe_count,pe_alloc_count,mda_count,dev_size
   * vgs --noheadings --units b --nosuffix --separator | --ignoreskippedcluster -o uuid,name,attr,size,free,extent_size,extent_count,free_count,tags,vg_mda_size, vg_mda_free,lv_count,pv_count,pv_name
   * lvs --noheadings --units b --nosuffix --separator | --ignoreskippedcluster -o uuid,name,vg_name,attr,size,seg_start_pe,devices,tags
   * lvcreate --autobackup n --contiguous [y|n] --size <size>m --addtag <tag> --name <lv name> <vg name>
   * lvremove -f --autobackup n <vg>/<lv>
   * lvextend --autobackup n --size <size>m <vg>/<lv>
   * lvreduce --autobackup n --size <size>m <vg>/<lv>
   * lvrename --autobackup n <vg> <old lv name> <new lv name>
   * lvchange --refresh <vg>/<lv>
   * lvchange --autobackup n --addtag <tag> <lv name>
   * lvchange --autobackup n --deltag <tag> <lv_name>
   * lvchange --autobackup n <vg>/<lv> --permission <permission>
   * lvchange --autobackup n \--deltag <del tag> \--addtag <add tag> <vg>/<lv>
     * Notes:
       * They keep a local in memory cache
 * targetd (uses python bindings in lvm2app)
   * lvm.vgOpen
   * vg.close
   * lvm.gc
   * vg.createLvThin
   * vg.createLvLinear
   * lv.remove
   * lv.snapshot
   * vg.lvFromName
   * vg.getSize
   * vg.getFreeSize
   * vg.getUuid
 * libvirt
   * lvcreate --name <name> -L <size>K --virtualsize <size>K -s <name> 
   * lvchange -aln <lv>
   * lvremove -f <lv>
   * lvs --separator # --noheadings --units b --unbuffered --nosuffix --options lv_name,origin,uuid,devices,segtype,stripes,seg_size,vg_extent_size,size,lv_attr <vg name>
 * clvmd (forks)
   * lvs  --config 'log{command_names=0 prefix=""}' --nolocking --noheadings -o vg_uuid,lv_uuid,lv_attr,vg_attr
 * dmeventd plugins - uses lvm2cmd.so
   * lvconvert --config devices{ignore_suspended_devices=1} --repair --use-policies $dev
   * lvscan --cache $dev
   * lvextend --use-policies $dev
 * collectd - https://github.com/collectd/collectd/blob/master/src/lvm.c
 * Heketi - https://github.com/heketi/heketi written in golang, appears to execute lvm command line over ssh
   * lvcreate --poolmetadatasize %vK -c 256K -L %vK -T %v/%v -V %vK -n %v"
   * lvs --options=lv_name,thin_count --separator=:"
   * lvremove -f %v/%v"
   * pvcreate --metadatasize=128M --dataalignment=256K %v"
   * vgcreate %v %v"
   * vgremove %v"
   * pvremove %v"
   * vgdisplay -c %v"

#### Command line usage summary (summary from data above)
 * PV operations
   * pvcreate
     * --dataalignment
   * pvresize
     * --setphysicalvolumesize
   * pvscan
     * --cache
   * pvremove
     * --force
     * --yes
   * pvmove
   * pvs
     * Options
     * --units [bk]
       * --nosuffix
       * --separator [:|]
       * --nameprefixes
       * --unquited
       * --noheadings
       * --ignoreskippedcluster
     * Columns
       * uuid
       * name
       * size
       * free
       * pv_name
       * pv_uuid
       * pe_start
       * pe_count
       * pe_alloc_count
       * mda_count
       * dev_size
       * vg_name
       * vg_uuid
       * vg_size
       * vg_free
       * vg_extent_size
       * vg_extent_count
       * vg_free_count
       * pv_count
 * VG Operations
   * vgcreate
     * -s
   * vgremove
     * --force
   * vgchange
     * -a y
     * -a n
   * vgreduce
     * --removemissing
     * --force
   * vgextend <vg> <pv>
   * vgs (with and without specifying vg)
     * Options
       * --version
       * --noheadings
       * --nosuffix
       * --nameprefixes
       * --unquoted 
       * --units [mgb]
       * --separator [:|]
       * --ignoreskippedcluster
     * Columns
       * name
       * attr
       * uuid
       * size
       * free
       * tags
       * extent_size
       * extent_count
       * free_count
       * pv_name
       * pv_count
       * lv_count
       * vg_name
       * vg_mda_size
       * vg_mda_free
   * vgrename
 * LV Operations
   * lvs (with and without specifying lv)
     * Options
       * -a
       * --separator [:|#]
       * --units [kgb]
       * --nosuffix
       * --nameprefixes
       * --unquoted
       * --noheadings
       * -o origin <vg_name>/<lv_name>
       * --ignoreskippedcluster
       * --unbuffered
       * --config
       * --nolocking
     * Columns
       * uuid
       * name
       * attr
       * size
       * seg_start_pe
       * devices
       * tags
       * vg_uuid
       * vg_attr
       * vg_name
       * vg_extent_size
       * segtype
       * seg_size
       * pool_lv
       * origin
       * stripes
       * data_percent
   * lvcreate
     * --autobackup n
     * -L <size>m
     * --size <size>[mb]
     * -n <lv_name>
     * --name
     * --snapshot
     * -y <vg_name>
     * -s
     * -n
     * --thinpool
     * --poolmetadatasize
     * --chunksize
     * --profile
     * --virtualsize
     * --contiguous
     * --addtag
     * -T
     * -V
     * -m
     * --nosync
     * --mirrorlog mirrored
     * -R
     * --permission
     * -r snapshot
   * lvremove
     * --autobackup n
     * --force
     * --yes
     * -f
     * --config
   * lvresize
     * --force
     * -L <size>[mb]
     * -r
   * lvchange
     * --autobackup n
     * --addtag
     * --deltag
     * -a y
     * -a n
     * --yes
     * -K
     * --refresh
     * --permission
     * -aln
   * lvconvert
     * --merge
   * lvrename
     * --autobackup n
   * lvdisplay
     * Options
     * --noheading
       * -C
     * Output
       * Attr
   * lvextend
     * --autobackup n
     * --size
     * -L
     * --use-policies
   * lvreduce
     * --autobackup n
     * --size
   * lvconvert
     * --config
     * --repair
     * --use-policies
   * lvscan
     * --cache

#### Some design possibilities
 * To  handle activation, a process is needed which can fork and lock memory  etc., thus having a separate daemon to handle such things is a  requirement.  So inter-process comunication between client and daemon is  needed.  Dbus is used and suggested by other teams as a good way to  achieve this.  DBus would provide and supports:
   * Language agnostic interface
   * Publish/subscribe interface for handling async. events
   * You  can also tell the UID, GID, PID and SELinux of DBus callers natively.  Authentication and access policy is done via polkit, in the absence of  polkit on a system you fall back to root -> allowed.
   * DBus  Signals. In addition if you want to only send signals to subscribed  callers, you can have a Subscribe() Unsubscribe() method similar to what  systemd does
   * DBus callers are identified by a unique caller name
   * Small number of library dependencies
     * libdbus, the lowest level DBus API requires little beyond libc, expat, and some security stuff like libselinux.
   * Self documenting through Introspection which is a big part of DBus
   * Handles versioning
     * DBus has multiple interfaces at multiple object paths, each with methods, properties and signals.
     * Adding methods to an interface does not break older callers, even function overloading with the same name.
     * New versioned interfaces can be added.
 * http://people.redhat.com/~teigland/.lvm-diagram3.jpg

#### Some proof of concept DBUS code is here: https://github.com/tasleson/lvm-dubstep

