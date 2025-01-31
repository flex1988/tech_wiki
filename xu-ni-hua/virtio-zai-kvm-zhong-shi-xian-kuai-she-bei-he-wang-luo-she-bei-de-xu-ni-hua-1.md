# virtio 在kvm中实现块设备和网络设备的虚拟化 - 1

## 1 virtio

virtio是一种IO（块设备和网络设备等）虚拟化技术，它是一种半虚拟化（paravirtualization）解决方案，由guest应用和hypervisor之间的一些IO通信协议组成，这也意味着设备驱动是专门为虚拟化设计的。同全虚拟化方案相比，半虚拟化方案大大减少了在guest和host中交互需要的cpu cycles，因为性能更加优秀。

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

## 2 Architecture

virtio可以分为这么几个部分：前端驱动层，后端驱动层和传输层

1. 前端virtio驱动，在guestos中一般作为一个设备驱动存在
2. 后端virtio驱动，在hypervisor中实现，接收前端virtio驱动的io 请求，在具体的物理设备上执行io请求
3. 传输层，前端virtio驱动和后端virtio驱动的通信通过一个称为virtqueue的队列来实现

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

## 3 前端virtio驱动

上图中的virtio-blk virtio-net virtio-pci virtio-balloon virtio-console都是前端virtio驱动。

这五种virtio设备可以抽象为virtio\_device，这样就可以通过virtqueue来与后端驱动通信。

这些数据类型定义在 linux/include/linux/virtio.h里。

```
/**
 * virtqueue - a queue to register buffers for sending or receiving.
 * @list: the chain of virtqueues for this device
 * @callback: the function to call when buffers are consumed (can be NULL).
 * @name: the name of this virtqueue (mainly for debugging)
 * @vdev: the virtio device this queue was created for.
 * @priv: a pointer for the virtqueue implementation to use.
 * @index: the zero-based ordinal number for this queue.
 * @num_free: number of elements we expect to be able to fit.
 *
 * A note on @num_free: with indirect buffers, each buffer needs one
 * element in the queue, otherwise a buffer will need one element per
 * sg element.
 */
struct virtqueue {
        struct list_head list;
        void (*callback)(struct virtqueue *vq);
        const char *name;
        struct virtio_device *vdev;
        unsigned int index;
        unsigned int num_free;
        void *priv;
};

/**
 * virtio_device - representation of a device using virtio
 * @index: unique position on the virtio bus
 * @failed: saved value for VIRTIO_CONFIG_S_FAILED bit (for restore)
 * @config_enabled: configuration change reporting enabled
 * @config_change_pending: configuration change reported while disabled
 * @config_lock: protects configuration change reporting
 * @dev: underlying device.
 * @id: the device type identification (used to match it with a driver).
 * @config: the configuration ops for this device.
 * @vringh_config: configuration ops for host vrings.
 * @vqs: the list of virtqueues for this device.
 * @features: the features supported by both driver and device.
 * @priv: private pointer for the driver's use.
 */
struct virtio_device {
        int index;
        bool failed;
        bool config_enabled;
        bool config_change_pending;
        spinlock_t config_lock;
        struct device dev;
        struct virtio_device_id id;
        const struct virtio_config_ops *config;
        const struct vringh_config_ops *vringh_config;
        struct list_head vqs;
        u64 features;
        void *priv;
};

/**
 * virtio_driver - operations for a virtio I/O driver
 * @driver: underlying device driver (populate name and owner).
 * @id_table: the ids serviced by this driver.
 * @feature_table: an array of feature numbers supported by this driver.
 * @feature_table_size: number of entries in the feature table array.
 * @feature_table_legacy: same as feature_table but when working in legacy mode.
 * @feature_table_size_legacy: number of entries in feature table legacy array.
 * @probe: the function to call when a device is found.  Returns 0 or -errno.
 * @scan: optional function to call after successful probe; intended
 *    for virtio-scsi to invoke a scan.
 * @remove: the function to call when a device is removed.
 * @config_changed: optional function to call when the device configuration
 *    changes; may be called in interrupt context.
 * @freeze: optional function to call during suspend/hibernation.
 * @restore: optional function to call on resume.
 */
struct virtio_driver {
        struct device_driver driver;
        const struct virtio_device_id *id_table;
        const unsigned int *feature_table;
        unsigned int feature_table_size;
        const unsigned int *feature_table_legacy;
        unsigned int feature_table_size_legacy;
        int (*validate)(struct virtio_device *dev);
        int (*probe)(struct virtio_device *dev);
        void (*scan)(struct virtio_device *dev);
        void (*remove)(struct virtio_device *dev);
        void (*config_changed)(struct virtio_device *dev);
#ifdef CONFIG_PM
        int (*freeze)(struct virtio_device *dev);
        int (*restore)(struct virtio_device *dev);
#endif
};
```

