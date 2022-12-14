
# kobject kset ktype uevent

[https://elixir.bootlin.com/linux/v5.10.72/source/include/linux/kobject.h#L64](https://elixir.bootlin.com/linux/v5.10.72/source/include/linux/kobject.h#L64)

```plain
struct kobject {
//Name, also appears at sysfs
//Use 'kobject_rename' to change name
    const char        *name;
//For adding to 'kset'
    struct list_head    entry;
//The parent kobject
    struct kobject        *parent;
//The kset. if no parent and kset is present.
//Set this kset to parent.(kset is special kobject)
    struct kset        *kset;
//The ktype, MUST have.
    struct kobj_type    *ktype;
    struct kernfs_node    *sd; /* sysfs directory entry */
//count
    struct kref        kref;
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    struct delayed_work    release;
#endif
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
//if 1, ignore the above report uevent.
    unsigned int uevent_suppress:1;
};
```

[https://elixir.bootlin.com/linux/v5.10.72/source/include/linux/kobject.h#L192](https://elixir.bootlin.com/linux/v5.10.72/source/include/linux/kobject.h#L192)

```plain
struct kset {
    struct list_head list;
//Protect list
    spinlock_t list_lock;
//Kset is a special kobject, also appears in sysfs
    struct kobject kobj;
    const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```

[https://elixir.bootlin.com/linux/v5.10.72/source/include/linux/kobject.h#L138](https://elixir.bootlin.com/linux/v5.10.72/source/include/linux/kobject.h#L138)

```plain
struct kobj_type {
//Callback for release kobject
    void (*release)(struct kobject *kobj);
//operation
    const struct sysfs_ops *sysfs_ops;
//kobject attribute
    struct attribute **default_attrs;    /* use default_groups instead */
    const struct attribute_group **default_groups;
    const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
    const void *(*namespace)(struct kobject *kobj);
    void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

![image-1668076709046.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-a10974098507622bb865c24f60bbf8e8.png?raw=true)

### ??????

1.  kobject ???????????????????????????????????????0??????????????????
2.  kobject memory??????????????????(??????????????????)
3.  kobject ???????????????kset???device\_driver???????????????????????????????????????/?????? (???kobject ??????)
4.  ktype ???release callback????????????kobject(and struct that include kobject) (???kset????????????ktype????????????????????????)
5.  uevent ???kobject??????????????????kobject??????????????????????????????user space (??????????????????????????????)

### kobject??????????????????sysfs folder

[https://elixir.bootlin.com/linux/v5.10.72/source/lib/kobject.c#L225](https://elixir.bootlin.com/linux/v5.10.72/source/lib/kobject.c#L225)

```plain
kobject *kobject_create_and_add(const char *name, struct kobject *parent)
    |
    |??????kobject_create() //???????????????????????????????????????
    |   |
    |   |??????kobject_init(kobj, &dynamic_kobj_ktype) //????????????ktype
    |
    |??????kobject_add(kobj, parent, "%s", name) //???????????????
        |
        |??????kobject_set_name_vargs(kobj, fmt, vargs) //??????kobject??????
        |
        |??????kobject_add_internal(kobj) //??????????????????
             |
             |??????kobject_get(kobj->parent) //??????parent???????????????????????????????????????kset??????parent
             |
             |??????kobj_kset_join(kobj) //?????????kset?????????
             |
             |??????create_dir(kobj) //????????????
???
static int create_dir(struct kobject *kobj)
    |
    |??????sysfs_create_dir_ns(kobj, kobject_namespace(kobj)) //???????????????kobj->name????????????
    |                                                       //????????????kobj->parent??????parent??????????????????
    |                                                       //???????????????kobj->ktype->namespace(kobj)
    |                                                       //?????????kernfs_node?????????kobj->sd???kernfs_node->priv??????kobj??????
    |
    |??????populate_dir(kobj) //???ktype????????????default_attrs??????????????????
    |   |
    |   |??????sysfs_create_file(kobj, attr)
    |
    |??????sysfs_create_groups(kobj, ktype->default_groups) //???ktype????????????default_groups??????????????????
    |
    |??????sysfs_get(kobj->sd) //???????????????????????????????????????????????????
    |
    |??????kobj_child_ns_ops(kobj) //????????????????????????????????????????????????kobj->parent->ktype->child_ns_type(kobj->parent)???
    |
    |??????sysfs_enable_ns(kobj->sd) //??????kobj?????????????????????
//https://zhuanlan.zhihu.com/p/464481582
```

![image-1668076735235.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-5dc8fdcd09b8b8522d97f4e447820bc5.png?raw=true)

### ??????kobject, kset, ktype

![image-1668076755278.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-8cb178721988a7b736396bf033e1f9d0.png?raw=true)

![image-1668076781123.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-fb382c2f9da4aeee3f261156371b38ba.png?raw=true)

1.  kobject???parent ???kset??????kobject
2.  kobject???kset
3.  kobject???ktype
4.  kset ??????????????????????????????kobject(??????????????????)
    
    ![image-1668076812115.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-24044d91ea50a269499a8df873581c1b.png?raw=true)
    

### Add kset to kset

![image-1668076825315.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-e79fe0a6e3ae769baeb2ebb328d12079.png?raw=true)

### connection

![image-1668076836074.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-c2c3a531297835cd7bce1f32e1e3829c.png?raw=true)

![image-1668076845095.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-4c9d88469a0b29c69268ea1bf74cf199.png?raw=true)

### connection explanation

kobject, kset ???driver model ?????????????????????????????????????????????????????????????????????????????????????????????????????????????????????struct??? ????????????struct????????????????????????????????????

```plain
struct device {
    struct kobject kobj;
    struct device       *parent;
???
    struct device_private   *p;
???
    const char      *init_name; /* initial name of the device */
    const struct device_type *type;
...
};
???
struct device_driver {
...
    struct driver_private *p;
};
???
struct driver_private {
    struct kobject kobj;
    struct klist klist_devices;
    struct klist_node knode_bus;
    struct module_kobject *mkobj;
    struct device_driver *driver;
};
```

driver ????????????????????????bus???kernel?????????driver????????????bus???kset??? device ??????match driver ???string???????????????probe ??????bus ???kset ??????????????????device ???drivers ???kset

```plain
root@edm-g-imx8mp:~# ls -la /sys/bus/i2c/
total 0
drwxr-xr-x  4 root root    0 Mar 24  2021 .
drwxr-xr-x 43 root root    0 Mar 24  2021 ..
drwxr-xr-x  2 root root    0 Mar 24  2021 devices
drwxr-xr-x 35 root root    0 Mar 24  2021 drivers
-rw-r--r--  1 root root 4096 Sep 19 09:38 drivers_autoprobe
--w-------  1 root root 4096 Sep 19 09:38 drivers_probe
--w-------  1 root root 4096 Mar 24  2021 uevent
???
??????i2c bus
??? ??????i2c???driver??????kset
??? ?????????i2c?????????device
  ??? ??????driver ??????match
    ??? device match driver??????, ??????device?????????kset??? 
      (device-tree ?????????????????????/sys/devices/platform/...????????????link)
```

```plain
# origin device tree
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/devices/                                                                                                                                                                                                                                                 
lrwxrwxrwx 1 root root 0 Mar 24  2021 1-001a -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-001a
lrwxrwxrwx 1 root root 0 Mar 24  2021 i2c-1 -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1
???
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/drivers/                                                                                                                                                                                                                                                 
drwxr-xr-x  2 root root 0 Mar 24  2021 wm8960
???
root@edm-g-imx8mp:~# ls -la /sys/bus/i2c/drivers/wm8960/
total 0
drwxr-xr-x  2 root root    0 Mar 24  2021 .
drwxr-xr-x 35 root root    0 Mar 24  2021 ..
lrwxrwxrwx  1 root root    0 Sep 19 09:35 1-001a -> ../../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-001a
--w-------  1 root root 4096 Sep 19 09:35 bind
--w-------  1 root root 4096 Mar 24  2021 uevent
--w-------  1 root root 4096 Sep 19 09:35 unbind
???
root@edm-g-imx8mp:/# ls -la ./sys/devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-001a/
total 0
drwxr-xr-x 3 root root    0 Mar 24  2021 .
drwxr-xr-x 8 root root    0 Mar 24  2021 ..
-r--r--r-- 1 root root 4096 Sep 19 09:36 consumers
lrwxrwxrwx 1 root root    0 Sep 19 09:36 driver -> ../../../../../../../bus/i2c/drivers/wm8960
-r--r--r-- 1 root root 4096 Sep 19 09:36 modalias
-r--r--r-- 1 root root 4096 Sep 19 09:34 name
lrwxrwxrwx 1 root root    0 Sep 19 09:36 of_node -> ../../../../../../../firmware/devicetree/base/soc@0/bus@30800000/i2c@30a30000/wm8960@1a
drwxr-xr-x 2 root root    0 Sep 19 09:36 power
lrwxrwxrwx 1 root root    0 Mar 24  2021 subsystem -> ../../../../../../../bus/i2c
-r--r--r-- 1 root root 4096 Sep 19 09:36 suppliers
-rw-r--r-- 1 root root 4096 Mar 24  2021 uevent
```

```plain
# miss-match wm8960 i2caddress???change to 1-0020???
root@edm-g-imx8mp:~# ls -la /sys/bus/i2c/devices/
lrwxrwxrwx 1 root root 0 Mar 24  2021 1-0020 -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-0020
lrwxrwxrwx 1 root root 0 Mar 24  2021 i2c-1 -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1
???
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/drivers/                                                                                                                                                                                                                                                 
drwxr-xr-x  2 root root 0 Mar 24  2021 wm8960
???
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/drivers/wm8960/                                                                                                                                                                                                                                          
total 0
drwxr-xr-x  2 root root    0 Sep 22 08:26 .
drwxr-xr-x 35 root root    0 Sep 22 08:26 ..
--w-------  1 root root 4096 Sep 22 08:30 bind
--w-------  1 root root 4096 Sep 22 08:26 uevent
--w-------  1 root root 4096 Sep 22 08:30 unbind
???
root@edm-g-imx8mp:/# ls -la ./sys/devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-0020                                                                                                                                                                                                   
total 0
drwxr-xr-x 3 root root    0 Sep 22 08:26 .
drwxr-xr-x 8 root root    0 Sep 22 08:26 ..
-r--r--r-- 1 root root 4096 Sep 22 08:28 consumers
-r--r--r-- 1 root root 4096 Sep 22 08:28 modalias
-r--r--r-- 1 root root 4096 Sep 22 08:26 name
lrwxrwxrwx 1 root root    0 Sep 22 08:28 of_node -> ../../../../../../../firmware/devicetree/base/soc@0/bus@30800000/i2c@30a30000/wm8960@20
drwxr-xr-x 2 root root    0 Sep 22 08:27 power
lrwxrwxrwx 1 root root    0 Sep 22 08:26 subsystem -> ../../../../../../../bus/i2c
-r--r--r-- 1 root root 4096 Sep 22 08:28 suppliers
-rw-r--r-- 1 root root 4096 Sep 22 08:26 uevent
# created by device-tree
```

### ??????

### ???????????????bus????????????![image-1668076860844.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-d18559ee7276c68b6e6397fe91ea1a48.png?raw=true)

[https://elixir.bootlin.com/linux/latest/source/drivers/base/base.h#L40](https://elixir.bootlin.com/linux/latest/source/drivers/base/base.h#L40)

```plain
struct subsys_private {
    struct kset subsys;
    struct kset *devices_kset;
    struct list_head interfaces;
    struct mutex mutex;
???
    struct kset *drivers_kset;
    struct klist klist_devices;
    struct klist klist_drivers;
    struct blocking_notifier_head bus_notifier;
    unsigned int drivers_autoprobe:1;
    struct bus_type *bus;
???
    struct kset glue_dirs;
    struct class *class;
};
```

??devices\_kset is in  
`/sys/devices/???`

### simple init code flow

[Linux ??????????????????????????????2???driver-bus-device???probe](https://www.cnblogs.com/schips/p/linux_device_model_2.html)

???????? : ???**????????????**

### pico-imx8mq example:

```plain
static int kobject_add_internal(struct kobject *kobj)
{
...
    pr_debug("kobject: '%s' (%p): %s: parent: '%s', set: '%s'\n",
         kobject_name(kobj), kobj, __func__,
         parent ? kobject_name(parent) : "<NULL>",
         kobj->kset ? kobject_name(&kobj->kset->kobj) : "<NULL>");
...
}
```

```plain
root@picoimx8mq:/# dmesg -t |grep kobject |grep "parent: '<NULL>'" |sort                                                                                                                                                                                                                          
kobject: 'block' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'bus' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'class' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'dev' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'devices' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'firmware' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'fs' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'kernel' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'module' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
kobject: 'power' ((____ptrval____)): kobject_add_internal: parent: '<NULL>', set: '<NULL>'
```

```plain
root@picoimx8mq:/# dmesg |grep kobjectls sys/
block  bus  class  dev  devices  firmware  fs  kernel  module  power
```

```plain
root@picoimx8mq:/# dmesg -t |grep kobject |grep "parent: 'bus'" |sort
kobject: 'amba' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'cec' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'clockevents' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'clocksource' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'container' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'coresight' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'cpu' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'edac' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'event_source' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'fsl-mc' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'genpd' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'gpio' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'hid' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'i2c' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'iio' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'mdio_bus' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'media' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'mipi-dsi' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'mmc' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'mmc_rpmb' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'node' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'nvmem' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'pci' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'pci-epf' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'pci_express' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'platform' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'rpmsg' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'scsi' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'sdio' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'serial' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'serio' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'soc' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'spi' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'spmi' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'tee' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'typec' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'ulpi' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'usb' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'usb-serial' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'virtio' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
kobject: 'workqueue' ((____ptrval____)): kobject_add_internal: parent: 'bus', set: 'bus'
```

```plain
root@picoimx8mq:/# ls /sys/bus/
amba         cpu           hid       mmc       pci_express  serio  ulpi
cec          edac          i2c       mmc_rpmb  platform     soc    usb
clockevents  event_source  iio       node      rpmsg        spi    usb-serial
clocksource  fsl-mc        mdio_bus  nvmem     scsi         spmi   virtio
container    genpd         media     pci       sdio         tee    workqueue
coresight    gpio          mipi-dsi  pci-epf   serial       typec
```

### Reference:

**Linux??????????????????(kobject, ?????????)** [https://pwl999.blog.csdn.net/article/details/78207820](https://pwl999.blog.csdn.net/article/details/78207820)

**kobject / kset / ktype???linux kernel ?????????????????????(code flow)** [https://zhuanlan.zhihu.com/p/464481582](https://zhuanlan.zhihu.com/p/464481582)

**???????????????kobject,kset???????????? (?????????, connection, connection explanation)** [https://blog.csdn.net/u011311586/article/details/51037219](https://blog.csdn.net/u011311586/article/details/51037219)

**Linux kobjects and sysfs filesystem(source code struct ???)** [https://blog.csdn.net/weixin\_41028621/article/details/101034234](https://blog.csdn.net/weixin_41028621/article/details/101034234)

**Linux ??????????????????????????????2???driver-bus-device???probe**(??????)\*\* [https://www.cnblogs.com/schips/p/linux\_device\_model\_2.html](https://www.cnblogs.com/schips/p/linux_device_model_2.html) (??????init ???????????????????????????)
