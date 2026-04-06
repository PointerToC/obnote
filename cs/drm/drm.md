# 前置知识--图像在内存中的表示
## 像素(Pixel)
像素是图像在离散采样网格上的一个最小单位，表示某一位置的光学/颜色信息的数值表示
## 色彩空间(Color Space)
抽象数学模型，通过一组数字来描述颜色，比如RGB或者YUV
## 位深(BPP，Bits Per Pixel)
每一个像素使用几位Bit来表达
以**ARGB8888**为例，一个像素在逻辑上由Red,Green,Blue,Alpha四个通道构成，在实际的物理内存存储中每个通道由8Bit表示，那么这样一个像素在内存中所占的大小为32Bit，即32BPP。
## 大小端
在内存里，红色（R）是在高位还是低位？这决定了像素在内存字节流中的真实顺序。
## 内存布局(Layout)
### 线性布局(linear layout)
将一张图像中的每一行像素在内存中连续存储。行与行之间通过pitch跳转。RGB图像格式的多采用这些布局。
```
内存：
+----------------------+
| row0                |
+----------------------+
| row1                |
+----------------------+
| row2                |
+----------------------+
```

### 多plane布局（multi-planar）

当图像的格式是YUV的时候，会采用这样的布局。
比如NV12(2 plane)
```
+----------------------+
| Y plane              |  ← offsets[0] = 0
+----------------------+
| UV plane             |  ← offsets[1]
+----------------------+
```
### 非线性布局
平铺(tiling),把图像切成小块（tile），再排列：
```
一个 tile（例如 4×4 像素）：

+----+----+
| p0 | p1 |
+----+----+
| p2 | p3 |
+----+----+
```

### 行步长(Pitch)
相邻两行在内存中的字节间距
一般而言**pitch ≥ width × bytes_per_pixel**,因为需要考虑内存对齐要求

# drm_framebuffer是什么？
本质是对内存中图像数据的描述，比如分辨率，像素信息和颜色格式等。他并不存储具体的图像数据，图像数据一律由其成员变量drm_gem_object内存对象存储。

结构体源码:
kernel/include/drm/drm_framebuffer.h
```
struct drm_framebuffer {
	// 该framebuffer所属的DRM设备
	struct drm_device *dev;
	// framebuffer的全局链表(dev->mode_config.fb_list)
	struct list_head head;
	// drm基类对象，oop思想
	struct drm_mode_object base;
	// 创建该 framebuffer 的进程名
	char comm[TASK_COMM_LEN];
	// 像素格式
	const struct drm_format_info *format;
	// 驱动提供的“虚函数表”，包含destroy，create_handle，dirty
	const struct drm_framebuffer_funcs *funcs;
	// 行步长(stride)
	unsigned int pitches[DRM_FORMAT_MAX_PLANES];
	// plane起始偏移
	// plane_base = buffer_base + offsets[i]
	unsigned int offsets[DRM_FORMAT_MAX_PLANES];
	// 内存布局类型
	uint64_t modifier;
	// 图像的宽度
	unsigned int width;
	// 图像的高度
	unsigned int height;
	int flags;
	unsigned int internal_flags;
	struct list_head filp_head;
	// 显存对象，真正存放数据的地方
	struct drm_gem_object *obj[DRM_FORMAT_MAX_PLANES];
};
```

## drm_gem_obj是谁申请的？里面的数据是由谁生成的？
### 申请者：kwin
调用逻辑：
![[gem_alloc.png]]

**GBM**:Generic Buffer Management.用户空间库，openGL和drm之间的桥梁。
**DRM_IOCTL_GEM_CREATE**由根据不同的芯片厂实现也不同。该函数的本质是分配一块内存，并返回对应的句柄
### 数据生产者：GPU
kwin调用渲染API,GPU将这些渲染出来的数据存放到对应的drm_gem_obj中的内存区域

**注意**：在这个时候，fb还未创建，有的只是一块gem内存和对应的数据

## 结合实际认识gem,fb,plane
```
[root@archlinux 1]# cat /sys/kernel/debug/dri/1/amdgpu_gem_info

pid     1091 command kwin_wayland:  
               0x00000009:     33423360 byte VRAM VISIBLE exported as ino:18 NO_CPU_ACCESS CPU_GTT_USWC VRAM_CLEARED VRAM_CONTIGUOUS   write fence:227:303432   signalled  
  
               0x0000000a:     33423360 byte VRAM VISIBLE pin count 1 exported as ino:20 NO_CPU_ACCESS CPU_GTT_USWC VRAM_CLEARED VRAM_CONTIGUOUS       write fence:227:303434   signalled 
```

```
[root@archlinux 1]# cat /sys/kernel/debug/dri/1/state
plane[201]: plane-3  
       crtc=crtc-0  
       fb=429  
               allocated by = kwin_wayland  
               refcount=2  
               format=AR30 little-endian (0x30335241)  
               modifier=0x200000000401b03  
               size=3840x2160  
               layers:  
                       size[0]=3840x2160  
                       pitch[0]=15360  
                       offset[0]=0  
                       obj[0]:  
                               name=0  
                               refcount=4  
                               start=00109d4e  
                               size=33423360  
                               imported=no  
       crtc-pos=3840x2160+0+0  
       src-pos=3840.000000x2160.000000+0.000000+0.000000  
       rotation=1  
       normalized-zpos=0  
       color-encoding=ITU-R BT.709 YCbCr  
       color-range=YCbCr limited range  
       color_mgmt_changed=0  
       color-pipeline=0
```

**amdgpu_gem_info**
0x0000000a:gem handle
pin count 1:表示当前gem被scanout,不能被替换

**drm state**
plane[201]:plane-3,当前plane为3的硬件图层在工作
crtc-0:当前plane被哪个crtc输出
fb=429:当前plane所绑定的fb id。fb中输出的字段跟drm_framebuffer结构体中的成员一一对应，其中obj[0]描述的就是gem_obj信息，size字段来看，跟0x0000000a的gem_obj大小一致。
从上述信息可知，0x0000000a的gem_obj被drm框架中id为429的fb封装，这个fb绑定到plane为3的硬件图层，同时该plane由crtc-0 scanout。输出格式为4k ARGB。

## 
# drm_plane
## drm_plane是什么？
显示控制器从内存中读取像素数据，在输出到屏幕之前，硬件会将多个图层（例如背景、视频层、鼠标光标）进行混合和缩放。drm框架中对该硬件模块的软件抽象就是drm_plane,每一个plane代表一个可独立配置的显示层。

结构体源码：
kernel/include/drm/drm_plane.h
```
struct drm_plane {
	// 该drm_plane所属的drm设备
	struct drm_device *dev;
    // plane链表节点
	struct list_head head;
	// plane名字，由drm分配
	char *name;
	struct drm_modeset_lock mutex;
	// drm基类对象，oop思想
	struct drm_mode_object base;
	// 表示该plane可以连接到哪个crtc上
	uint32_t possible_crtcs;
	// 硬件支持的所有颜色格式
	uint32_t *format_types;
	unsigned int format_count;
	// 兼容老的驱动而存在的字段,在没有上报所支持的颜色格式的情况下分配一个默认值
	bool format_default;
	// 对应支持的数据布局格式
	uint64_t *modifiers;
	unsigned int modifier_count;
	// 兼容老驱动而存在的字段
	struct drm_crtc *crtc;
	struct drm_framebuffer *fb;
	struct drm_framebuffer *old_fb;
	// 基础操作函数，update_plane, disable_plane, destroy
	const struct drm_plane_funcs *funcs;
	// 通用属性容器，暴露给用户空间
	struct drm_object_properties properties;
	// plane的类型:primary,overlay,cursor
	enum drm_plane_type type;
	// plane在系统的索引
	unsigned index;
	// 辅助函数
	const struct drm_plane_helper_funcs *helper_private;
	// 现代支持atomic事务的架构核心，保存plane硬件在某一时刻的‘快照’
	struct drm_plane_state *state;
	// 透明度
	struct drm_property *alpha_property;
	// 图层的上下层级顺序
	struct drm_property *zpos_property;
	// 旋转/翻转
	struct drm_property *rotation_property;
	// 混合模式，用于图层叠加
	struct drm_property *blend_mode_property;
	// YUV格式下颜色的转换公式和范围
	struct drm_property *color_encoding_property;
	struct drm_property *color_range_property;
	struct drm_property *scaling_filter_property;
	// 光标位图中，“真正起作用的点”距离左/右边缘的像素距离
	struct drm_property *hotspot_x_property;
	struct drm_property *hotspot_y_property;
	// 保存崩溃信息
	struct kmsg_dumper kmsg_panic;
};
```
## 关注的重点
### drm_plane_type
```
// kernel/include/drm/drm_plane.h
enum drm_plane_type {
	DRM_PLANE_TYPE_OVERLAY,
	DRM_PLANE_TYPE_PRIMARY,
	DRM_PLANE_TYPE_CURSOR,
};
```
primary:主平面，每个crtc必须绑定一个主平面，通常该图层跟显示器的分辨率一致，可以理解为显示器的底图
cursor:鼠标平面，专门给鼠标使用的图层
overlay:除了主平面和鼠标平面之外的平面

#### 结合实际认知drm_plane_type
```
[root@archlinux 1]# cat /sys/kernel/debug/dri/1/state
plane[201]: plane-3  
       crtc=crtc-0  
       fb=429  
               allocated by = kwin_wayland  
               refcount=2  
               format=AR30 little-endian (0x30335241)  
               modifier=0x200000000401b03  
               size=3840x2160  
               layers:  
                       size[0]=3840x2160  
                       pitch[0]=15360  
                       offset[0]=0  
                       obj[0]:  
                               name=0  
                               refcount=4  
                               start=00109d4e  
                               size=33423360  
                               imported=no  
       crtc-pos=3840x2160+0+0  
       src-pos=3840.000000x2160.000000+0.000000+0.000000  
       rotation=1  
       normalized-zpos=0  
       color-encoding=ITU-R BT.709 YCbCr  
       color-range=YCbCr limited range  
       color_mgmt_changed=0  
       color-pipeline=0
plane[360]: plane-6  
       crtc=crtc-0  
       fb=428  
               allocated by = kwin_wayland  
               refcount=2  
               format=AR24 little-endian (0x34325241)  
               modifier=0x0  
               size=256x256  
               layers:  
                       size[0]=256x256  
                       pitch[0]=1024  
                       offset[0]=0  
                       obj[0]:  
                               name=0  
                               refcount=4  
                               start=001419d6  
                               size=262144  
                               imported=no  
       crtc-pos=256x256+670+1908  
       src-pos=256.000000x256.000000+0.000000+0.000000  
       rotation=1  
       normalized-zpos=1  
       color-encoding=ITU-R BT.601 YCbCr  
       color-range=YCbCr limited range  
       color_mgmt_changed=0  
       color-pipeline=0

[root@archlinux 1]# drm_info | grep -A 20 "Plane"

   ├───Plane 6  
   │   ├───Object ID: 360  
   │   ├───CRTCs: {0}  
   │   ├───Legacy info  
   │   │   ├───FB ID: 428  
   │   │   │   ├───Object ID: 428  
   │   │   │   ├───Size: 256×256  
   │   │   │   ├───Format: ARGB8888 (0x34325241)  
   │   │   │   ├───Modifier: DRM_FORMAT_MOD_LINEAR (0x0000000000000000)  
   │   │   │   └───Planes:  
   │   │   │       └───Plane 0: offset = 0, pitch = 1024 bytes  
   │   │   └───Formats:  
   │   │       └───ARGB8888 (0x34325241)  
   │   └───Properties  
   │       ├───"type" (immutable): enum {Overlay, Primary, Cursor} = Cursor  
   │       ├───"FB_ID" (atomic): object framebuffer = 428  
   │       │   ├───Object ID: 428  
   │       │   ├───Size: 256×256  
   │       │   ├───Format: ARGB8888 (0x34325241)  
   │       │   ├───Modifier: DRM_FORMAT_MOD_LINEAR (0x0000000000000000)  
   │       │   └───Planes:  
   │       │       └───Plane 0: offset = 0, pitch = 1024 bytes  
   │       ├───"IN_FENCE_FD" (atomic): srange [-1, INT32_MAX] = -1  
   │       ├───"CRTC_ID" (atomic): object CRTC = 363  
   │       ├───"CRTC_X" (atomic): srange [INT32_MIN, INT32_MAX] = 670  
   │       ├───"CRTC_Y" (atomic): srange [INT32_MIN, INT32_MAX] = 1908  
   │       ├───"CRTC_W" (atomic): range [0, INT32_MAX] = 256  
   │       ├───"CRTC_H" (atomic): range [0, INT32_MAX] = 256  
   │       ├───"SRC_X" (atomic): range [0, UINT32_MAX] = 0  
   │       ├───"SRC_Y" (atomic): range [0, UINT32_MAX] = 0  
   │       ├───"SRC_W" (atomic): range [0, UINT32_MAX] = 256  
   │       ├───"SRC_H" (atomic): range [0, UINT32_MAX] = 256  
   │       ├───"IN_FORMATS" (immutable): blob = 361  
   │       │   └───DRM_FORMAT_MOD_LINEAR (0x0000000000000000)  
   │       │       └───ARGB8888 (0x34325241)  
   │       └───"zpos" (immutable): range [UINT8_MAX, UINT8_MAX] = 255
```
由上述输出可知这是一个primary+cursor叠加的场景
根据
```
crtc-pos=3840x2160+0+0
```
可知plane-3为primary，为整个显示界面的底图
根据
```
"type" (immutable): enum {Overlay, Primary, Cursor} = Cursor
```
可知plane为cursor图层，cursor图层的一般大小也只有256\*256。
同时，将鼠标移动到不同的位置，其crtc-pos属性也会发生相应的变化，比如：
```
crtc-pos=256x256+670+1908 ---> crtc-pos=256x256+23+90 
```
