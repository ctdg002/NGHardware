#### Synchronization & Cache Control

Fence: QueueSubmit 所有CB complete
Semaphore: 跨queue的ResourceAccess 
Event: 在queue内的CB间、CB与cpu同步
PipeLineBarrier: CB内同步
RenderPass: 同步框架

##### Execution and Memory Dependencies

###### Execution Sync: 同步执行，不保证原子内存操作，只保证原子IF
+ Synchronization Scope: 一个syncCB可以构造同步依赖的对象
   + specific pipeline stage(Mask), 隐式包含logically-earlier(As)/logically-after(Bs)
   + specific CB
+ Execution Dependency: 仅对SyncScope中的指令起作用
   + submit order: A S B
   + execute order: As Bs
+ Execution Dependency Chain: 一串syncCB
   + syncCB1: As --> Bs
   + syncCB2: Bs --> Cs

###### Memory Sync: 原子内存操作
data hazard: read-after-write, ==write-after-write==
+ 内存同步操作
   + AvailiabilityOp：确保SyncScope内的写操作可以被 某个MemoryDomain 读取
   + ==MemoryDomainOp: 确保写入AMemoryDomain的内存可以被BMemoryDomain读取，通常在cpu/gpu间切换==
      + QueueSubmission 从cpu到gpu
      + vkFlushMappedMemoryRanges 从gpu到cpu
   + VisibilityOp: 允许 某个MemoryDomain 读取
+ Memory Dependency（Chain）: Execution Dependency（Chain）中含有AvailabilityOp/VisibilityOp
   + Instruction(As && 1st access scope) -> AvailableOp(srcAccessMask)
   + Availiable -- layout transition --> Visible
   + VisibleOp(dstAccessMask) -> Instruction(Bs && 2nd access scope)
+ Memory Access Scope: 
   + first access scope: srcAccessMask应用的SyncScope
   + second access scope: dstAccessMask应用的SyncScope

###### Image Layout Transition
+ Memory Bound: 内存导致的pipeline stall占比
+ image subresource range: ==VkImageAspectFlagBits==, mipRange, 2DArrayLayer
+ image layout transition: layout1 -- 可能执行内部RW --> layout2
   + old layout == ==VK_IMAGE_LAYOUT_UNDEFIND(内存可能被丢弃)==/current layout subresource range
   + ==当存在SIMULTANEOUS bounded resources时，gpu会做memory aliasing，这时可能执行内部RW==
   + new layout
   + queue上 所有layout transition CBs严格按照CB submit顺序执行

###### SyncScope: Pipeline stages
+ Pipeline Stage: 由Action/Sync CB决定
   + 跨Pipeline Stage的CB必须遵守implicit ordering guarantees
   + TOP_OF_PIPE/BOTTOM_OF_PIPE: accessFlag = 0
   + DRAW_INDIRECT(可在QUEUE_COMPUTE)/VERTEX_INPUT: VBO consumed
   + EARLY_FRAGMENT_TEST: subpass load DS + earlyZ
   + Xxx_SHADER: shader storage
   + LATE_FRAGMENT_TEST: ZTest + subpass store DS
   + COLOR_ATTACHMENT_OUTPUT: final output
   + TRANSFER: Copy/Blit/Resolve/Clear
       + QUEUE_GRAPHICS/COMPUTE/TRANSFER
   + HOST: cpu may RW
   + ALL_GRAPHICS
   + ALL_COMMANDS
+ (logically) Pipeline Stage Order: 一种implicit ordering guarantee
   + logical earlier在logical after前init
   + 驱动不支持的SyncScopePipeStage会被理解为：
      + As：替换为支持同步的logical later PipeStage
      + Bs: 替换为支持同步的logical earlier PipeStage
      + 不会导致overlap，但会导致不必要的stall
   + QUEUE_GRAPHICS: IA -> VS -> TCS -> TES -> GS -> EARLY-Z -> PS -> ZTest -> OUTPUT
   + QUEUE_COMPUTE: IA -> CS
   + QUEUE_TRANSFER: Transfer
+ Sync CB的执行不由Pipeline Stage Order决定


###### Access Type
+ ==AccessScope: 指定Type的access via SyncScope==
+ VkAccessFlags
    + INDIRECT_COMMAND_READ: DRAW_INDIRECT
    + INDEX_READ:VERTEX_INPUT
    + VERTEX_ATTR_READ:VERTEX_INPUT
    + UNIFORM_READ:SHADER
    + ==INPUT_ATTACHMENT_READ: FRAGMENT_SHADER==
    + SHADER_RW:SHADER
       + 可读：
          + uniform/uniform texel/sampled image
          + storage/physical storage/storage texel/storage image
       + 可写：以上带storage的
    + ==COLOR_ATTACHMENT_RW:COLOR_ATTACHMENT_OUTPUT==
       + 可读：可能通过blendOp/logicOp/subpassLoad获取
       + 可写：Output/Resolve/subpassStore
    + ==DS_ATTACHMENT_RW: EARLY_FRAGMENT_TEST/LATE_FRAGMENT_TEST==
    + TRANSFER_RW：
       + 可读：copy
       + 可写：clear/copy
    + HOST_RW: 直接在内存上操作
    https://blog.csdn.net/weiaipan1314/article/details/95662392
    https://www.zhihu.com/question/399443653
       + 先锁GPU DDR地址,CPU再读取
          + 如无MEMORY_PROPERTY_HOST_COHERENT（cpu&gpu共享内存）, 在gpu正确的MemoryDependency后，为执行上述过程需要调用vkFlushMappedMemoryRanges
       + CPU先写入内存，锁地址，GPU再读取
          + vkQueueSubmit隐式执行上述过程
    + MEMORY_RW: 以上所有

###### FrameBuffer Region Dependency
+ framebuffer region SyncScope: [X,Y,ArrayLayer, RasterSample]
   + 当且仅当sample不同时，执行像素粒度的SyncScope匹配
   + 当且仅当sample相同时，执行sample粒度的SyncScope匹配
   + ==[X,Y]必须小于等于WARP_SIZE==
   + 例如绑定的InputAttachment与FrameBuffer的sample不对应时，执行像素粒度的MemoryBarrier，此后的PS可以获取像素的==任意sample==;当InputAttachment与FrameBuffer的sample对应，且==逐sample作FrameBuffer-Local的MemoryBarrier==时，此后的PS仅可以获取像素的==对应sample==
      + 猜测：大三角形场景的Resolve一定要与绘制分subpass做
+ framebuffer-space PipelineStage: 
FRAGMENT_SHADER/EARLY_FRAGMENT_TESTS/LATE_FRAGMENT_TESTS/COLOR_ATTACHMENT_OUTPUT
+ 当且仅当这些PS的Dependency会转化为FrameBuffer-Global或多个FrameBuffer-Local
   + FrameBuffer-Local需要设置DEPENDENCY_BY_REGION,且==不会影响non-framebuffer-space PS==
   + FrameBuffer-Global会导致所有WARP一起FlushData

###### View-Local Dependency
+ multiview(VR或多视图)会需要view-local和view-global同步
+ view-local需要设置DEPENDENCY_VIEW_LOCAL

###### Device-Local Dependency
+ Device-Local: 逐physical-device同步
+ subpass的所有PassInfo::deviceMask参与同步
+ pipelineBarrier的所有CB::deviceMask参与同步
+ non-device-local需要设置VK_DEPENDENCY_DEVICE_GROUP
+ ==semaphore和event只在一个physical device上执行==

##### Implicit Synchronization Guarantee


###### 使用相关
https://zhuanlan.zhihu.com/p/161619652
https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples

只读：Execution Dependency
读写：Memory Dependency


