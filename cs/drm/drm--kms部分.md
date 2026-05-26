**简介**
drm(Direct Rendering Manager,Linux 内核图形显示架构)有如下几大核心子系统：GPU driver,KMS,GEM/TTM,
**kms**
Kernel Mode Setting，简单来说，
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

# drm_framebuffer
本质是对内存中图像数据的描述，比如分辨率，像素信息和颜色格式等。它并不存储具体的图像数据，图像数据一律由其成员变量drm_gem_object内存对象存储。

结构体源码:
```
// kernel/include/drm/drm_framebuffer.h

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

![](gem_create.svg)
**GBM**:Generic Buffer Management。用户空间库，openGL和drm之间的桥梁。
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

# drm_plane
显示控制器中存在多个“并行图像输入通道”，每个通道都能独立从内存读取像素，并在硬件中通过缩放、色彩转换和 alpha 混合实时合成为最终被输出。drm框架中对这个“并行图像输入通道”的抽象就是drm_plane。简而言之，一个plane对应一个图像输入通道。一个drm实例中，drm_plane的数量和能力和硬件一致。

结构体源码：
```
// kernel/include/drm/drm_plane.h

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
## drm_plane中关注的重点
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
![](plane_type.png)
#### 为什么要分多个plane type?
**假设：只有一个plane，一个fb**

![](one_plane.svg)

在此场景下，gpu除了需要参与渲染之外，还需要参与合成工作。此外，桌面中任一窗口的变动，gpu都重新绘制整个桌面，每一次的重新绘制gpu都需要读一次应用缓存，合成之后再写回一次fb，内存读写开销巨大。
比如，现在正在窗口化播放一个视频，显示器输出帧率为4k60。那么，gpu每秒钟的内存读写带宽为：
```
3840*2160*4*60*2 = 3.8GB/S 
```

**在多plane的设计下**

![](multi_plane.svg)

视频数据在解码后直接提交至overlay plane,不需要gpu进行颜色格式的转换。相对于单一plane的设计，多plane将原本需要gpu开销的任务，统一由对应的plane硬件单元并行处理。极大地减小了gpu的开销。

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
### drm_plane_state
drm_plane结构体中的大多成员用于描述一个plane的能力，是静态的。而drm_plane_state则是描述当前plane在某一时刻完整配置状态的'快照',是动态的。随着每次画面的更新而改变。它是atomic kms架构中的核心数据结构。
```
struct drm_plane_state {
	struct drm_plane *plane;
	// 当前输出到的crtc
	struct drm_crtc *crtc;
	// 当前绑定的fb
	struct drm_framebuffer *fb;
	// 同步，等待fence出发后再提交事务
	struct dma_fence *fence;
	// fb中截取的区域和输出到crtc的区域
	int32_t crtc_x;
	int32_t crtc_y;
	uint32_t crtc_w, crtc_h;
	uint32_t src_x;
	uint32_t src_h, src_w;
	int32_t hotspot_x, hotspot_y;
	// 当前状态下对像素的处理信息
	u16 alpha;
	uint16_t pixel_blend_mode;
	unsigned int rotation;
	unsigned int zpos;
	unsigned int normalized_zpos;
	enum drm_color_encoding color_encoding;
	enum drm_color_range color_range;
	struct drm_property_blob *fb_damage_clips;
	bool ignore_damage_clips;
	struct drm_rect src, dst;
	bool visible;
	enum drm_scaling_filter scaling_filter;
	struct drm_crtc_commit *commit;
	struct drm_atomic_state *state;
	bool color_mgmt_changed : 1;
};
```

# drm_crtc
CRTC（Cathode Ray Tube Controller）本质是一个**扫描引擎+时序生成器**，它从fb中扫描扫描数据，然后生成显示时序，同时把这些时序输出到encoder。一般而言，一个crtc对应一路独立的显示输出。drm_crtc就是对该硬件的抽象。

```
struct drm_crtc {
	struct drm_device *dev;
	struct device_node *port;
	struct list_head head;
	char *name;
	struct drm_modeset_lock mutex;
	struct drm_mode_object base;
	// 兼容legacy API
	struct drm_plane *primary;
	struct drm_plane *cursor;
	unsigned index;
	int cursor_x;
	int cursor_y;
	bool enabled;
	// 用户请求的模式
	struct drm_display_mode mode;
	// 经过驱动转换后的实际硬件时序
	struct drm_display_mode hwmode;
	int x;
	int y;
	// 核心操作回调函数
	const struct drm_crtc_funcs *funcs;
	uint32_t gamma_size;
	uint16_t *gamma_store;
	// 辅助操作回调函数
	const struct drm_crtc_helper_funcs *helper_private;
	// 属性追踪，暴露给用户
	struct drm_object_properties properties;
	struct drm_property *scaling_filter_property;
	// 原子状态
	struct drm_crtc_state *state;
	// 追踪尚未完成的原子提交请求的链表
	struct list_head commit_list;
	spinlock_t commit_lock;
	struct dentry *debugfs_entry;
	struct drm_crtc_crc crc;
	unsigned int fence_context;
	spinlock_t fence_lock;
	unsigned long fence_seqno;
	char timeline_name[32];
	struct drm_self_refresh_data *self_refresh_data;
};
```

## crtc关注重点
### drm_display_mode 显示模式与时序

![](display_mode.png)
朴素的理解显示器显示图像：就是crtc以一个像素点为单位，从左到右、从上到下地发送信号，这些信号代表每个RGB分量的值，显示器根据这些值来点亮对应的像素点从而显示图像。
#### 信号的同步
pixel clock时钟，每个时钟周期输出一个像素
#### 怎么知道这一行输出结束？
hsync信号，每发出一次脉冲，就代表这一行输出结束，该信号叫做行同步信号
#### 怎么知道这一帧输出结束？
vsync信号，每发出一次脉冲，就代表这一帧输出结束，该信号叫做帧同步信号
#### 同步极性
水平同步信号(hsync)和垂直同步信号(vsync)在触发同步时的电平逻辑状态。
#### 前沿和后沿(front porch,back porch)
**front porch:**
hsync信号之前，一行的像素流结束之后，等待硬件进入准备同步的时间
**back porch:**
hsync信号之后，确保了在第一颗像素数据发送之前，同步信号已经完全结束
#### active area
有效像素区域
#### Blanking Interval (消隐区)
front porch + sync + back porch的总和
**把mode展开，如图所示**

```
               Active                 Front           Sync           Back
             Region                 Porch                          Porch
    <-----------------------><----------------><-------------><-------------->
       //////////////////////|
      ////////////////////// |
     //////////////////////  |..................               ................
                                               _______________
    <----- [hv]display ----->
    <------------- [hv]sync_start ------------>
    <--------------------- [hv]sync_end --------------------->
    <-------------------------------- [hv]total ----------------------------->*

```
由上述图片可知：
```
hsync_start = hdisplay + hfront
hsync_end   = hsync_start + hsync
htotal      = hsync_end + hback
vsync_start = vdisplay + vfront
vsync_end   = vsync_start + vsync
vtotal      = vsync_end + vback
```

# drm_encoder
encoder是对drm pipeline中的信号转换硬件的抽象。它负责将crtc输出的通用时序信号转换为panel所需要的时序信号。
在这样的设计下，将两者的功能解耦。crtc只负责输出通用的视频信号，encoder负责将通用时序信号转换为对应的协议。
以输出hdmi信号为例：
```
CRTC 输出：  
Pixel stream (RGB/YUV)  
+ HSync / VSync / DE  
+ Pixel Clock  
  
↓  
  
HDMI Encoder：  
1. 数据重组（Data Island / Video）  
2. TMDS 编码（8b/10b-like）  
3. 串行化（Serializer）  
4. 差分输出（TMDS lanes）

↓  

panel
```
**源码结构体**
```
struct drm_encoder {
	struct drm_device *dev;
	struct list_head head;
	struct drm_mode_object base;
	char *name;
	// 当前转换的协议，如dp,hdmi等
	int encoder_type;
	unsigned index;
	uint32_t possible_crtcs;
	uint32_t possible_clones;
	struct drm_crtc *crtc;
	struct list_head bridge_chain;
	// 不同厂商根据自己的硬件能力实现的回调函数
	const struct drm_encoder_funcs *funcs;
	const struct drm_encoder_helper_funcs *helper_private;
	struct dentry *debugfs_entry;
};
```
**关注重点：struct drm_encoder_helper_funcs \*helper_private**
以rk3588为例，在驱动注册时候会根据硬件的能力集，注册对应的回调函数
```
rockchip_drm_init()
	rockchip_dp_probe()
		rockchip_dp_bind()
			rockchip_drm_create_encoder()
				drm_encoder_helper_add(encoder, &rockchip_dp_encoder_helper_funcs)
```
```
static struct drm_encoder_helper_funcs rockchip_dp_encoder_helper_funcs = {
	.mode_valid = rockchip_dp_drm_encoder_mode_valid,
	.mode_fixup = rockchip_dp_drm_encoder_mode_fixup,
	.mode_set = rockchip_dp_drm_encoder_mode_set,
	.atomic_enable = rockchip_dp_drm_encoder_enable,
	.atomic_disable = rockchip_dp_drm_encoder_disable,
	.atomic_check = rockchip_dp_drm_encoder_atomic_check,
};

static enum drm_mode_status
rockchip_dp_drm_encoder_mode_valid(struct drm_encoder *encoder,
				   const struct drm_display_mode *mode)
{
	struct rockchip_dp_device *dp = to_dp(encoder);
	struct videomode vm;

	drm_display_mode_to_videomode(mode, &vm);

	if (!vm.hfront_porch || !vm.hback_porch || !vm.vfront_porch || !vm.vback_porch) {
		DRM_DEV_ERROR(dp->dev, "front porch or back porch can not be 0\n");
		return MODE_BAD;
	}

	return MODE_OK;
}
```
其中mode_valid回调函数显示：当前edp模块不支持front porch/back porch为0的时序。因此，当tcon端需要front porch/back porch为0的时序时，drm pipeline会在这里停止而导致输出中止，从而出现老化屏的现象。

在arm soc架构中，不需要太多关注encoder这一层。原因在bridge章节中叙述。

# drm_bridge
是一种将视频信号转换成另外一种视频信号的硬件模块，位于encoder和最终显示设备之间，可以无限级联。

结构体源码
```
struct drm_bridge {
	struct drm_private_obj base;
	struct drm_device *dev;
	// 当前bridge所连接的encoder
	struct drm_encoder *encoder;
	// 多级bridge链表
	struct list_head chain_node;
#ifdef CONFIG_OF
	struct device_node *of_node;
#endif
	struct list_head list;
	// 时序
	const struct drm_bridge_timings *timings;
	// 芯片控制操作函数集，由不同厂商实现，重要
	const struct drm_bridge_funcs *funcs;
	void *driver_private;
	// 位掩码，用来表示这个桥接器支持哪些操作（如检测、EDID读取、HPD等）
	enum drm_bridge_ops ops;
	// 输出接口类型
	int type;
	bool interlace_allowed;
	struct i2c_adapter *ddc;
	struct mutex hpd_mutex;
	void (*hpd_cb)(void *data, enum drm_connector_status status);
	void *hpd_data;
};
```
## 不同显示架构下，bridge和encoder的差异
bridge和encoder在硬件上都属于信号转换器。在amd gpu的架构中，drm_bridge是drm pipeline中非必要的组件，属于可选的一部分。在arm soc中，encoder 是“CRTC → 输出接口类型”的抽象锚点；bridge 是“信号链上的具体变换模块”，bridge为必带的组件。

以输出hdmi信号为例：
amd gpu的drm pipeline为：
```
crtc->encoder->connector->panel
```
rk3588的drm pipeline为：
```
crtc->encoder->bridge(synopsys)->connector->panel
```
**同样的将标准时序信号转换为hdmi信号，为什么在amd gpu中这个实际转换者为encoder,而在rk3588中，这个实际转换者为bridge?**
pc领域的gpu从设计之初就是高集成度的单体架构。但是在arm soc架构中，这些信号转换硬件都是第三方的ip核，为了降低代码耦合，便将这部分的工作移到了bridge。

## encoder只是crtc和bridge之间的路由，bridge才是核心逻辑载体
以rk3588为例，hdmitx和dp/edp中对应的driver controller在drm pipeline中被封装成drm_bridge而不是drm_encoder。
### edp
源码位置
```
kernel/drivers/gpu/drm/bridge/analogix/*
```
驱动probe:
```
rockchip_dp_probe():dp->adp = analogix_dp_probe()
	analogix_dp_prove():dp->bridge.funcs = &analogix_dp_bridge_funcs
```
edp controller核心回调函数全部在analogix_dp_bridge_funcs结构体中，比如atomic_enable函数，edp的链路训练，协商，信号输出都封装在这个回调函数中。
不同tx驱动的在这个回调函数结构体中实现自己的核心控制逻辑，然后将bridge连接到drm pipeline中。
```
static const struct drm_bridge_funcs analogix_dp_bridge_funcs = {
	.atomic_duplicate_state = drm_atomic_helper_bridge_duplicate_state,
	.atomic_destroy_state = drm_atomic_helper_bridge_destroy_state,
	.atomic_reset = drm_atomic_helper_bridge_reset,
	.atomic_pre_enable = analogix_dp_bridge_atomic_pre_enable,
	.atomic_enable = analogix_dp_bridge_atomic_enable,
	.atomic_disable = analogix_dp_bridge_atomic_disable,
	.atomic_post_disable = analogix_dp_bridge_atomic_post_disable,
	.attach = analogix_dp_bridge_attach,
	.detach = analogix_dp_bridge_detach,
	.mode_valid = analogix_dp_bridge_mode_valid,
};
```
### edp的bridge又是怎么加入到drm pipeline中的？
rk drm是个聚合驱动，先probe各个子组件，待所有组件资源准备好之后，统一调用各个子组件注册的bind回调函数，在bind函数中将bridge,encoder,connector绑定到drm pipeline中
```
rockchip_dp_bind()
|	rockchip_dp_drm_create_encoder(dp);
|	analogic_dp_bind(dp->adp, drm_dev);
|--> analogix_dp_bind(dp->adp, drm_dev)
     |-->analogix_dp_bridge_init()
		|-->drm_bridge_attach()
			|	bridge->dev = encoder->dev; //绑定bridge和encoder
			|	bridge->encoder = encoder;
			|-->bridge->funcs->attach(bridge, flags)
					// 初始化connector
					ret = drm_connector_init(dp->drm_dev, connector,
					 &analogix_dp_connector_funcs,
					 connector_type);
					 drm_connector_helper_add(connector,
					 &analogix_dp_connector_helper_funcs);
					 // 绑定connector和encoder
					 drm_connector_attach_encoder(connector, encoder);
					 // 绑定connector和bridge
					 ret = dp->plat_data->attach(dp->plat_data, bridge, connector);
```
详细阅读下rockchip_dp_drm_create_encoder()和analogix_dp_bridge_attach()这个两个函数可以加深理解rk drm中encoder和bridge的区别。

hdmitx的probe,bind,attach逻辑结构和上述一致。
# drm_connector
drm_connector是对物理显示接口的抽象。

结构体源码
```
struct drm_connector {
	struct drm_device *dev;
	struct device *kdev;
	struct device_attribute *attr;
	struct list_head head;
	struct drm_mode_object base;
	char *name;
	struct mutex mutex;
	unsigned index;
	int connector_type;
	int connector_type_id;
	bool interlace_allowed;
	bool doublescan_allowed;
	bool stereo_allowed;
	bool ycbcr_420_allowed;
	enum drm_connector_registration_state registration_state;
	struct list_head modes;
	enum drm_connector_status status;
	struct list_head probed_modes;
	struct drm_display_info display_info;
	const struct drm_connector_funcs *funcs;
	struct drm_property_blob *edid_blob_ptr;
	struct drm_object_properties properties;
	struct drm_property *scaling_mode_property;
	struct drm_property *vrr_capable_property;
	struct drm_property *colorspace_property;
	struct drm_property_blob *path_blob_ptr;
	struct drm_property *max_bpc_property;

#define DRM_CONNECTOR_POLL_HPD (1 << 0)
#define DRM_CONNECTOR_POLL_CONNECT (1 << 1)
#define DRM_CONNECTOR_POLL_DISCONNECT (1 << 2)

	uint8_t polled;
	int dpms;
	const struct drm_connector_helper_funcs *helper_private;
	struct drm_cmdline_mode cmdline_mode;
	enum drm_connector_force force;
	bool override_edid;
	u64 epoch_counter;
	u32 possible_encoders;
	struct drm_encoder *encoder;

#define MAX_ELD_BYTES	128

	uint8_t eld[MAX_ELD_BYTES];
	bool latency_present[2];
	int video_latency[2];
	int audio_latency[2];
	struct i2c_adapter *ddc;
	int null_edid_counter;
	unsigned bad_edid_counter;
	bool edid_corrupt;
	u8 real_edid_checksum;
	struct dentry *debugfs_entry;
	struct drm_connector_state *state;
	struct drm_property_blob *tile_blob_ptr;
	bool has_tile;
	struct drm_tile_group *tile_group;
	bool tile_is_single_monitor;
	uint8_t num_h_tile, num_v_tile;
	uint8_t tile_h_loc, tile_v_loc;
	uint16_t tile_h_size, tile_v_size;
	struct llist_node free_node;
	struct hdr_sink_metadata hdr_sink_metadata;
};

```
## 连接状态的改变与热插拔监测
```
enum drm_connector_status {
	connector_status_connected = 1,
	connector_status_disconnected = 2,
	connector_status_unknown = 3,
};

enum drm_connector_status status;
```
status表示当前物理接口和panel的连接状态，正常情况下，当我们在shell输入：
```
cat /sys/class/drm/card1-DP-1/status
```
查询当前panel和connector之间的状态的时候，查询的就是这个值
**也可以通过修改这个值的状态来打开和关闭整个显示链路**
```
echo on > /sys/class/drm/card1-DP-1/status
echo off > /sys/class/drm/card1-DP-1/status
```
### 主动打开显示链路的逻辑：
核心函数
```
kernel/drivers/gpu/drm/drm_sysfs.c

static ssize_t status_store(struct device *device,
			   struct device_attribute *attr,
			   const char *buf, size_t count)
{
	...
	if (sysfs_streq(buf, "detect"))
		connector->force = 0;
	else if (sysfs_streq(buf, "on"))
		connector->force = DRM_FORCE_ON;
	else if (sysfs_streq(buf, "on-digital"))
		connector->force = DRM_FORCE_ON_DIGITAL;
	else if (sysfs_streq(buf, "off"))
		connector->force = DRM_FORCE_OFF;
	else
		ret = -EINVAL;

	if (old_force != connector->force || !connector->force) {
		DRM_DEBUG_KMS("[CONNECTOR:%d:%s] force updated from %d to %d or reprobing\n",
			      connector->base.id,
			      connector->name,
			      old_force, connector->force);

		connector->funcs->fill_modes(connector,
					     dev->mode_config.max_width,
					     dev->mode_config.max_height);
	}
	...
}
```

`fill_modes`回调，绝大部分厂商用的都是drm框架自带的`drm_helper_probe_connector_modes`函数。
函数的大致逻辑如下：
```
kernel/drivers/gpu/drm/drm_probe_helper.c

int drm_helper_probe_single_connector_modes(struct drm_connector *connector,
					    uint32_t maxX, uint32_t maxY)
{
	...
	
	// 当echo on时，这里直接强制修改状态，不走硬件探测
	// 这里的force和drm_helper_probe_detect函数都会执行厂商自定的回调
	if (connector->force) {
		if (connector->force == DRM_FORCE_ON ||
		    connector->force == DRM_FORCE_ON_DIGITAL)
			connector->status = connector_status_connected;
		else
			connector->status = connector_status_disconnected;
		if (connector->funcs->force)
			connector->funcs->force(connector);
	} else {
		ret = drm_helper_probe_detect(connector, &ctx, true);

		if (ret == -EDEADLK) {
			drm_modeset_backoff(&ctx);
			goto retry;
		} else if (WARN(ret < 0, "Invalid return value %i for connector detection\n", ret))
			ret = connector_status_unknown;

		connector->status = ret;
	}

	// 如果当前状态和之前的状态不一样，在这里会唤醒一个异步任务output_poll_work，该异步任务会执行drm_kms_helper_hotplug_event来通知用户层连接状态的改变
	
	if (old_status != connector->status) {
		dev->mode_config.delayed_event = true;
		if (dev->mode_config.poll_enabled)
			mod_delayed_work(system_wq,
					 &dev->mode_config.output_poll_work,
					 0);
	}
	
	...
	
	// 读edid逻辑，执行厂商自定的get_modes回调
	count = (*connector_funcs->get_modes)(connector);
	
	...
	
	// 模式合法校验,在这里会执行所有组件在bind阶段注册的mode_valid回调
	list_for_each_entry(mode, &connector->modes, head) {
		if (mode->status != MODE_OK)
			continue;

		mode->status = drm_mode_validate_driver(dev, mode);
		if (mode->status != MODE_OK)
			continue;

		mode->status = drm_mode_validate_size(mode, maxX, maxY);
		if (mode->status != MODE_OK)
			continue;

		mode->status = drm_mode_validate_flag(mode, mode_flags);
		if (mode->status != MODE_OK)
			continue;
			
		ret = drm_mode_validate_pipeline(mode, connector, &ctx,
						 &mode->status);
						 
		if (mode->status != MODE_OK)
			continue;
		mode->status = drm_mode_validate_ycbcr420(mode, connector);
	}

prune:

	...
		
	// 模式排序，选出prefered mode
	drm_mode_sort(&connector->modes);

	list_for_each_entry(mode, &connector->modes, head) {
		drm_mode_set_crtcinfo(mode, CRTC_INTERLACE_HALVE_V);
		// 输出所有合法的mode
		drm_mode_debug_printmodeline(mode);
	}

	return count;
}
```