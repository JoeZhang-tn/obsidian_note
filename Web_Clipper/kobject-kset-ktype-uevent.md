
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

### 理解

1.  kobject 主要功能為”存一個數值，為0時自動釋放”
2.  kobject memory都為動態註冊(才能動態釋放)
3.  kobject 通常存在於kset與device\_driver中，所以這些都必須動態註冊/釋放 (由kobject 完成)
4.  ktype 中release callback負責釋放kobject(and struct that include kobject) (同kset可以不同ktype只是操作上要小心)
5.  uevent 是kobject的一部分，當kobject狀態發生改變時，通知user space (只要使用於熱插拔裝置)

### kobject⇒管理並建立sysfs folder

[https://elixir.bootlin.com/linux/v5.10.72/source/lib/kobject.c#L225](https://elixir.bootlin.com/linux/v5.10.72/source/lib/kobject.c#L225)

```plain
kobject *kobject_create_and_add(const char *name, struct kobject *parent)
    |
    |——kobject_create() //申请内存并初始化结构体变量
    |   |
    |   |——kobject_init(kobj, &dynamic_kobj_ktype) //关联默认ktype
    |
    |——kobject_add(kobj, parent, "%s", name) //添加进系统
        |
        |——kobject_set_name_vargs(kobj, fmt, vargs) //设置kobject名称
        |
        |——kobject_add_internal(kobj) //加入文件系统
             |
             |——kobject_get(kobj->parent) //增加parent引用计数，如果没有则会使用kset作为parent
             |
             |——kobj_kset_join(kobj) //加入到kset链表中
             |
             |——create_dir(kobj) //创建目录
​
static int create_dir(struct kobject *kobj)
    |
    |——sysfs_create_dir_ns(kobj, kobject_namespace(kobj)) //创建名称为kobj->name的目录，
    |                                                       //父目录为kobj->parent，无parent则挂在根目录
    |                                                       //命名空间为kobj->ktype->namespace(kobj)
    |                                                       //返回的kernfs_node保存在kobj->sd，kernfs_node->priv指向kobj地址
    |
    |——populate_dir(kobj) //为ktype中的所有default_attrs创建对应文件
    |   |
    |   |——sysfs_create_file(kobj, attr)
    |
    |——sysfs_create_groups(kobj, ktype->default_groups) //为ktype中所有的default_groups创建对应文件
    |
    |——sysfs_get(kobj->sd) //增加对目录节点的引用，避免其被销毁
    |
    |——kobj_child_ns_ops(kobj) //获取命名空间的操作方法（获取源：kobj->parent->ktype->child_ns_type(kobj->parent)）
    |
    |——sysfs_enable_ns(kobj->sd) //使能kobj目录的命名空间
//https://zhuanlan.zhihu.com/p/464481582
```

![image-1668076735235.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-5dc8fdcd09b8b8522d97f4e447820bc5.png?raw=true)

### 圖解kobject, kset, ktype

![image-1668076755278.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-8cb178721988a7b736396bf033e1f9d0.png?raw=true)

![image-1668076781123.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-fb382c2f9da4aeee3f261156371b38ba.png?raw=true)

1.  kobject的parent 為kset中的kobject
2.  kobject的kset
3.  kobject的ktype
4.  kset 會紀錄所有屬於自己的kobject(最後變為下圖)
    
    ![image-1668076812115.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-24044d91ea50a269499a8df873581c1b.png?raw=true)
    

### Add kset to kset

![image-1668076825315.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-e79fe0a6e3ae769baeb2ebb328d12079.png?raw=true)

### connection

![image-1668076836074.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-c2c3a531297835cd7bce1f32e1e3829c.png?raw=true)

![image-1668076845095.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-4c9d88469a0b29c69268ea1bf74cf199.png?raw=true)

### connection explanation

kobject, kset 是driver model 的基本結構體，使用這兩個做層次關係描述，但實際很少碰到，因為已經被包在其他大型struct中 （用大型struct的其他成員表示具體功能）

```plain
struct device {
    struct kobject kobj;
    struct device       *parent;
​
    struct device_private   *p;
​
    const char      *init_name; /* initial name of the device */
    const struct device_type *type;
...
};
​
struct device_driver {
...
    struct driver_private *p;
};
​
struct driver_private {
    struct kobject kobj;
    struct klist klist_devices;
    struct klist_node knode_bus;
    struct module_kobject *mkobj;
    struct device_driver *driver;
};
```

driver 首先必須屬於一個bus，kernel會將此driver加入到該bus的kset中 device 如果match driver 的string，就會進行probe 所以bus 的kset 中，通常會有device 及drivers 的kset

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
​
找到i2c bus
├ 將此i2c的driver放進kset
└ 找到此i2c的所有device
  └ 所有driver 進行match
    └ device match driver的話, 將此device加到此kset中 
      (device-tree 模式下都會創建/sys/devices/platform/...，但不會link)
```

```plain
# origin device tree
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/devices/                                                                                                                                                                                                                                                 
lrwxrwxrwx 1 root root 0 Mar 24  2021 1-001a -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-001a
lrwxrwxrwx 1 root root 0 Mar 24  2021 i2c-1 -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1
​
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/drivers/                                                                                                                                                                                                                                                 
drwxr-xr-x  2 root root 0 Mar 24  2021 wm8960
​
root@edm-g-imx8mp:~# ls -la /sys/bus/i2c/drivers/wm8960/
total 0
drwxr-xr-x  2 root root    0 Mar 24  2021 .
drwxr-xr-x 35 root root    0 Mar 24  2021 ..
lrwxrwxrwx  1 root root    0 Sep 19 09:35 1-001a -> ../../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-001a
--w-------  1 root root 4096 Sep 19 09:35 bind
--w-------  1 root root 4096 Mar 24  2021 uevent
--w-------  1 root root 4096 Sep 19 09:35 unbind
​
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
# miss-match wm8960 i2caddress（change to 1-0020）
root@edm-g-imx8mp:~# ls -la /sys/bus/i2c/devices/
lrwxrwxrwx 1 root root 0 Mar 24  2021 1-0020 -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1/1-0020
lrwxrwxrwx 1 root root 0 Mar 24  2021 i2c-1 -> ../../../devices/platform/soc@0/30800000.bus/30a30000.i2c/i2c-1
​
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/drivers/                                                                                                                                                                                                                                                 
drwxr-xr-x  2 root root 0 Mar 24  2021 wm8960
​
root@edm-g-imx8mp:/# ls -la /sys/bus/i2c/drivers/wm8960/                                                                                                                                                                                                                                          
total 0
drwxr-xr-x  2 root root    0 Sep 22 08:26 .
drwxr-xr-x 35 root root    0 Sep 22 08:26 ..
--w-------  1 root root 4096 Sep 22 08:30 bind
--w-------  1 root root 4096 Sep 22 08:26 uevent
--w-------  1 root root 4096 Sep 22 08:30 unbind
​
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

### 總結

### 以其中一個bus作為例子![image-1668076860844.png](https://github.com/JoeZhang-tn/obsidian_note_image/blob/main/1668675604-d18559ee7276c68b6e6397fe91ea1a48.png?raw=true)

[https://elixir.bootlin.com/linux/latest/source/drivers/base/base.h#L40](https://elixir.bootlin.com/linux/latest/source/drivers/base/base.h#L40)

```plain
struct subsys_private {
    struct kset subsys;
    struct kset *devices_kset;
    struct list_head interfaces;
    struct mutex mutex;
​
    struct kset *drivers_kset;
    struct klist klist_devices;
    struct klist klist_drivers;
    struct blocking_notifier_head bus_notifier;
    unsigned int drivers_autoprobe:1;
    struct bus_type *bus;
​
    struct kset glue_dirs;
    struct class *class;
};
```

 devices\_kset is in  
`/sys/devices/…`

### simple init code flow

[Linux 内核：设备驱动模型（2）driver-bus-device与probe](https://www.cnblogs.com/schips/p/linux_device_model_2.html)

 章節 : “**初始化”**

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

**Linux驱动设备模型(kobject, 結構圖)** [https://pwl999.blog.csdn.net/article/details/78207820](https://pwl999.blog.csdn.net/article/details/78207820)

**kobject / kset / ktype（linux kernel 中的面向对象）(code flow)** [https://zhuanlan.zhihu.com/p/464481582](https://zhuanlan.zhihu.com/p/464481582)

**设备模型之kobject,kset及其关系 (結構圖, connection, connection explanation)** [https://blog.csdn.net/u011311586/article/details/51037219](https://blog.csdn.net/u011311586/article/details/51037219)

**Linux kobjects and sysfs filesystem(source code struct 圖)** [https://blog.csdn.net/weixin\_41028621/article/details/101034234](https://blog.csdn.net/weixin_41028621/article/details/101034234)

**Linux 内核：设备驱动模型（2）driver-bus-device与probe**(總結)\*\* [https://www.cnblogs.com/schips/p/linux\_device\_model\_2.html](https://www.cnblogs.com/schips/p/linux_device_model_2.html) (詳細init 及註冊流程可以這篇)
