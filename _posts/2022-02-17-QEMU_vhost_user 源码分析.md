---
layout: post
title: QEMU_vhost_user 源码分析
date: 2022-02-17
tags: jekyll
---

## 概述

qemu 提供 vhost user特性，相比virtio设备，提升了性能。相比vhost设备，提升了灵活性。通过qemu与其他进程共享内存，完成数据的传输。通过unix socket，实现qemu 与第三方进程间通信。

## vhost user 设备

### 初始化

从qemu角度看，所有的vhost user设备，都会实现init，realize。然后创建一个基础设备结构。在kernel枚举设备阶段，对相应的vhost user设备进行激活。接着就可以按需使用该设备功能。

**init**

qemu设备的init过程分为两个阶段，设备对应的class类init()和instance实例化对象init()。

类init过程：

```
hw\block\vhost-user-blk.c
vhost_user_blk_class_init
	// 创建device class和virtio device class
	DeviceClass *dc = DEVICE_CLASS(klass);
    VirtioDeviceClass *vdc = VIRTIO_DEVICE_CLASS(klass);
    
    // 注册realize回调函数
    vdc->realize = vhost_user_blk_device_realize;
    vdc->unrealize = vhost_user_blk_device_unrealize;
    vdc->get_config = vhost_user_blk_update_config;
    vdc->set_config = vhost_user_blk_set_config;
    vdc->get_features = vhost_user_blk_get_features;
    vdc->set_status = vhost_user_blk_set_status;
    vdc->reset = vhost_user_blk_reset;
```

对象实例化过程：

```
vhost_user_blk_instance_init
	VHostUserBlk *s = VHOST_USER_BLK(obj);
	device_add_bootindex_property(obj, &s->bootindex, "bootindex", "/disk@0,0", DEVICE(obj));
```

**realize**

当上一步init完成后，开始向vhost user blk设备设置参数值，例如队列数量和大小，初始化vhost user设备。

```
hw\block\vhost-user-blk.c
vhost_user_blk_device_realize
	// 每个vhost user设备都必须指定chardev设备
	if (!s->chardev.chr)
		return;
	
	// 参数校验 s->num_queues，s->queue_size
	...
	
	// 初始化vhost user设备
	vhost_user_init(&s->vhost_user, &s->chardev, errp)
	// 初始化virtio设备
	virtio_init(vdev, "virtio-blk", VIRTIO_ID_BLOCK, sizeof(struct virtio_blk_config));
	// 创建virtio队列
	for (i = 0; i < s->num_queues; i++) {
        s->virtqs[i] = virtio_add_queue(...);
    }
    
    qemu_chr_fe_set_handlers(&s->chardev,  NULL, NULL, vhost_user_blk_event,
                             NULL, (void *)dev, NULL, true);
```

上述步骤vhost_user_init，主要作用是向vhost user设备保存chardev。

```
hw\virtio\vhost-user.c
vhost_user_init
	user->chr = chr;
    user->memory_slots = 0;
```

virtio_init初始化，将vhost user blk当做一个virtio blk设备，向guest OS注册。对于虚拟机内部，看到的始终是一个virtio blk设备。但是在数据处理过程中，已经由第三方app接管数据处理流程。

注意，上述qemu_chr_fe_set_handlers()函数，当qemu启动后，dpdk会与qemu建立vhost socket链接，调用vhost_user_blk_event回调函数。

```
vhost_user_blk_event
	 switch (event)
	 	case CHR_EVENT_OPENED:
	 		// 连接vhost user blk
	 		vhost_user_blk_connect(dev)
	 		
vhost_user_blk_connect
	// 设置参数
	s->connected = true;
    s->dev.nvqs = s->num_queues;
    s->dev.vqs = s->vhost_vqs;
    s->dev.vq_index = 0;
    s->dev.backend_features = 0;
    
    // 设置vhost user设备init
    vhost_dev_init(VHOST_BACKEND_TYPE_USER)
    	// 根据入参是user类型，设置ops回调函数
    	vhost_set_backend_type(hdev, backend_type)
    		dev->vhost_ops = &user_ops;
    	// 按照流程，调用对应的ops，该流程兼容vhost，vhost user，vdpa
    	hdev->vhost_ops->vhost_backend_init
    	hdev->vhost_ops->vhost_set_owner
    	hdev->vhost_ops->vhost_get_features
    	...
    	// 当region变化时，回调保存或删除region信息
    	memory_listener_register(&hdev->memory_listener, &address_space_memory)
```

**vhost user ops**

上文提到，根据设备类型，注册对应的ops回调函数。接着按照流程，调用相应的函数。在vhost user中，对应的回调函数如下：

```
const VhostOps user_ops = {
        .backend_type = VHOST_BACKEND_TYPE_USER,
        .vhost_backend_init = vhost_user_backend_init,
		...
        .vhost_set_mem_table = vhost_user_set_mem_table,
        .vhost_set_vring_addr = vhost_user_set_vring_addr,
        .vhost_set_vring_endian = vhost_user_set_vring_endian,
        .vhost_set_vring_num = vhost_user_set_vring_num,
        .vhost_set_vring_base = vhost_user_set_vring_base,
        .vhost_get_vring_base = vhost_user_get_vring_base,
        .vhost_set_vring_kick = vhost_user_set_vring_kick,
        .vhost_set_vring_call = vhost_user_set_vring_call,
        .vhost_set_features = vhost_user_set_features,
        .vhost_get_features = vhost_user_get_features,
        .vhost_set_owner = vhost_user_set_owner,
        .vhost_reset_device = vhost_user_reset_device,
        ...
};
```

### 通信协议

**vhost user msg 格式**

vhost user 采用的是unix本地socket通信。对于第三方app，也需要遵循socket通信机制。在设备初始化过程中，需要对上述的每个ops回调函数对相应的处理。

对于协议本身，也需要定义结构体，保存协议请求类型，标记位，数据长度，数据载体等。在qemu，用VhostUserMsg保存

```
typedef struct VhostUserMsg {
    VhostUserHeader hdr;
    VhostUserPayload payload;
} QEMU_PACKED VhostUserMsg;
```

msg包含hdr和payload两部分

```
typedef struct {
    VhostUserRequest request;
    uint32_t flags;
    uint32_t size; /* the following payload size */
} QEMU_PACKED VhostUserHeader;

typedef union {
        uint64_t u64;
        struct vhost_vring_state state;
        struct vhost_vring_addr addr;
        VhostUserMemory memory;
        VhostUserMemRegMsg mem_reg;
        VhostUserLog log;
        struct vhost_iotlb_msg iotlb;
        VhostUserConfig config;
        VhostUserCryptoSession session;
        VhostUserVringArea area;
        VhostUserInflight inflight;
} VhostUserPayload;
```

hdr又包括requet请求类型，flags版本号，是否需要reply信息，size表示payload长度。payload作为一个union类型，可以保存virtio队列状态，地址，或者内存qemu内存信息。

当qemu需要把协商数据发送到第三方进程时，首先要将数据封装。以vhost_user_set_vring_num()函数为例：

```
vhost_user_set_vring_num
	vhost_set_vring(dev, VHOST_USER_SET_VRING_NUM, ring)
		// 设置hdr和payload数据
		VhostUserMsg msg = {
        	.hdr.request = request,
        	.hdr.flags = VHOST_USER_VERSION,
        	.payload.state = *ring,
        	.hdr.size = sizeof(msg.payload.state),
    	};
    	// 发送数据
    	vhost_user_write(dev, &msg, NULL, 0)
```

**socket 格式**

在vhost_user_write函数内部调用如下：

```
vhost_user_write
	// 发送fds
	qemu_chr_fe_set_msgfds(chr, fds, fd_num)
	// 发送msg
	qemu_chr_fe_write_all(chr, (const uint8_t *) msg, size)
```

连接的backend是channel socket，所以调用过程如下：

```
qemu_chr_fe_set_msgfds
	// 保存文件句柄
	set_msgfds -> tcp_set_msgfds
		memcpy(s->write_msgfds, fds, num * sizeof(int))
		s->write_msgfds_num = num

qemu_chr_fe_write_all
	// 向socket写入msg数据
	qemu_chr_write
		...
			// io\channel-socket.c
			io_writev -> qio_channel_socket_writev
				msg.msg_iov = (struct iovec *)iov;
    			msg.msg_iovlen = niov;
    			// 如果需要传递文件句柄
    			if (nfds) {
                  	 msg.msg_control = control;
        			msg.msg_controllen = CMSG_SPACE(sizeof(int) * nfds);
        			cmsg = CMSG_FIRSTHDR(&msg);
        			cmsg->cmsg_len = CMSG_LEN(fdsize);
        			cmsg->cmsg_level = SOL_SOCKET;
        			cmsg->cmsg_type = SCM_RIGHTS;
        			memcpy(CMSG_DATA(cmsg), fds, fdsize);
    			}
    			// 调用系统接口，向unix socket写入
				sendmsg(sioc->fd, &msg, 0)	
```

在qemu端，使用sendmsg/recvmsg发送或接受数据。会使用msghdr结构体保存传输数据。该结构在内核中定义如下：

```
struct msghdr {
   void         *msg_name;       /* optional address */
   socklen_t     msg_namelen;    /* size of address */
   struct iovec *msg_iov;        /* scatter/gather array */
   size_t        msg_iovlen;     /* # elements in msg_iov */
   void         *msg_control;    /* ancillary data, see below */
   size_t        msg_controllen; /* ancillary data buffer len */
   int           msg_flags;      /* flags on received message */
};

struct iovec {               /* Scatter/gather array items */
   	void  *iov_base;         /* Starting address */
   	size_t iov_len;          /* Number of bytes to transfer */
};
```

iovec用来保存数据起始地址和长度。msg_control用于指向与协议控制相关的消息或者辅助数据。例如为了保存fds文件句柄数据，填充到cmsghdr结构体，然后msg_control指向该结构体。

```
struct cmsghdr {
      size_t cmsg_len;    /* Data byte count, including header (type is socklen_t in POSIX) */
      int    cmsg_level;  /* Originating protocol */
      int    cmsg_type;   /* Protocol-specific type */
      /* followed by unsigned char cmsg_data[]; */
};
```

## 数据面-内存共享

**获取共享内存信息**

通信机制上，通过unix socket连接qemu和第三方进程数据交换。但是所有数据都通过socket通信方式传递，会造成IO性能差。所以，和virtio机制一样，将guest 与 qemu的内存共享机制应用到guest与第三方进程内存共享。

基本原理是：guest 在产生数据后，向virtio queue写入。qemu将这段virtio queue共享内存区域，以mem-share形式，即存在后端内存文件/匿名文件。然后又映射给第三方进程。这时候，第三方进程可以直接从这段匿名文件，也就是对应的virtio queue共享内存区域取出数据。

```
hw\virtio\vhost.c
vhost_dev_init
	// 注册保存虚拟机内存ram信息
	memory_listener_register(&hdev->memory_listener, &address_space_memory);
		listener_add_address_space(listener, as)
			listener->region_add(listener, &section) -> vhost_region_addnop
```

经过一系列调用，用于保存内存信息的数据结构 VhostUserMemoryRegion 被填充，内容分别是每个region对应的虚拟机物理地址(gpa)，内存长度，进程虚拟地址(hva)，在region区域做了mmap映射的偏移地址。

```
typedef struct VhostUserMemoryRegion {
    uint64_t guest_phys_addr;
    uint64_t memory_size;
    uint64_t userspace_addr;
    uint64_t mmap_offset;
} VhostUserMemoryRegion;
```

**传递共享内存信息**

该过程是将上述获取的共享内存信息发送到第三方进程，由第三方进程做mmap

```
hw\block\vhost-user-blk.c
vhost_user_blk_connect
	vhost_user_blk_start
		vhost_dev_start
			hdev->vhost_ops->vhost_set_mem_table(hdev, hdev->mem)
```

vhost_set_mem_table函数主要作用是保存上述的内存结构体四个值，然后将该数据发送给第三方进程

```
hw\virtio\vhost-user.c
vhost_user_set_mem_table
	vhost_user_fill_set_mem_table_msg(u, dev, &msg, fds, &fd_num, false)
	vhost_user_write(dev, &msg, fds, fd_num)

vhost_user_fill_set_mem_table_msg
	// 如果是内存共享，即mem-share
	if (fd > 0)
		vhost_user_fill_msg_region
			dst->userspace_addr = src->userspace_addr;
    		dst->memory_size = src->memory_size;
    		dst->guest_phys_addr = src->guest_phys_addr;
    		dst->mmap_offset = mmap_offset;
```

## 管理面-eventfd

基本原理是：当guest需要发送通知notify，就会eventfd写1。在qemu启动阶段，会调用vhost_set_vring_call()和vhost_set_vring_kick()，把ioeventfd和中断irqfd向第三方进程注册。由第三方进程epoll ioeventfd。guest向eventfd写1，第三方进程立刻就可以轮询到，不再经过qemu进程。接着从virtio queue取数据，执行数据处理操作，最后发送中断。

```
hw\block\vhost-user-blk.c
vhost_user_blk_start
	hw\virtio\vhost.c
	vhost_dev_start
		vhost_virtqueue_start
			// 获取host notify event fd
			file.fd = event_notifier_get_fd(virtio_queue_get_host_notifier(vvq));
			// 向第三方进程注册host notify guest eventfd 
			dev->vhost_ops->vhost_set_vring_kick(dev, &file)
			// 向第三方进程注册guest notify host eventfd
			dev->vhost_ops->vhost_set_vring_call(dev, &file)
```

如文章上面内容，当guest 发送notify时，第三方进程可以直接epoll。当第三方进程处理完I/O数据，可以直接写call的eventfd，qemu可以epoll到事件，然后调用qemu的中断处理函数，进一步向kvm发送ioctl，最终向KVM注入中断。

## 参考

1.https://www.man7.org/linux/man-pages/man3/cmsg.3.html