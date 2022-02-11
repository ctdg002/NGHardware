#### Initialization 创建&获取feature support
VkCmds + VkInstance
##### FunctionPointer
注意由版本带来的alias，UTF-8
获取函数指针：vkGetInstanceProcAddr(Instance,pName)
返参类型：PFN_vkVoidFunction
调用函数的vkObj: 、
   + VkInstance/Vk(Physical)Device/VkQueue/VkCommandBuffer
   + 函数指针可能内部根据非Instance的vkObj做Dispatch
   + 可以通过具体获取函数指针的避免
      + vkGetDeviceProcAddr
      + PhysicalDevice仅可通过vkGetInstanceProcAddr
          + 枚举可用特性：vkGetPhysicalDeviceProperties、vkEnumerateDeviceExtensionProperties
###### Instance
+ vkCreateInstance/vkDestroyInstance
   + 创建时根据app信息初始化VkLibrary implementation
   + 传参VkInstanceCreateInfo::ppEnabledExtensionNames获取必要扩展
   + 传参VkInstanceCreateInfo::ppEnabledLayerNames数组元素按host->driver排列
   + 需求版本未被implementation支持时返回VK_ERROR_INCOMPATIBLE_DRIVER

```
VkInstanceCreateInfo{
    sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    pNEXT = NULL,
    flags = 0,
    //XxxNames = UTF-8 string,
}
```
https://www.khronos.org/registry/vulkan/specs/1.2-khr-extensions/html/vkspec.html#LoaderAndLayerInterface
https://github.com/KhronosGroup/Vulkan-Loader/blob/master/loader/LoaderAndLayerInteface.md

+ vkEnumerateInstanceVersion
   + 枚举Instance信息，不会分配资源，但loader/layer可能导致OOM

https://www.khronos.org/registry/vulkan/specs/1.2-khr-extensions/html/vkspec.html#extendingvulkan-coreversions-versionnumbers

+ instance&physicalInstance必须externally synchronized

#### Devices and Queues

##### Physical Device GPU硬件相关特性
+ 获取特性 = Enumerate + GetProperties
+ out pProperties 
   + vendorID: GPU chipset/SOC accelerator
      + [PCI](https://pcisig.com/membership/member-companies) 0x0... 
      + Khronos 0x1.. @ vk.xml/vulkan_core.h
   + deviceID: 
      + DeviceLUID(locally unique identifier to adapter)
      + DeviceUUID(universally... to dirver)
+ PHYSICAL_DEVICE_TYPE: integrated, discrete, virtual, cpu
+ Vk1.1一些重要的DeviceProperties
   + subgroupSize == wavefront ?
   + subgroupSupportedStages = VK_QUEUE_GRAPHICS_BIT / VK_QUEUE_COMPUTE_BIT
   + pointClippingBehavior点基元裁剪策略
   + maxMultiviewViewCount/maxPerSetDescriptors
   + protectedNoFault 内存保护机制/maxMemoryAllocationSize
+ Vk1.2一些重要的DeviceProperties
   + conformance test version 接口一致性检测
   + 浮点数运算规则  
      + signedZeroInfNanPreserve
      + denormFlushToZero
      + roundToEven/Zero
   + 可更新Descriptor数（PerStage...）
   + （uniform） buffer是否支持unordered access，不支持时动态合批必须按顺序以避免反复遍历
   + sampler数组、texture数组是否支持unordered access
   + robust buffer access 根据Info::range进行buffer边界检查、指针正确性检查
   + quadDivergentImplicitLOD 非quad一致的mipmap采样
   + supportedResolveMode
      + Depth
      + Stencil
      + IndependentResolve：Depth/Stencil方式不同
   + 支持mipmap的format，mipmap方式
   + semaphore最大差别
   + samples per pixel数
##### Logical Device 逻辑特性,state & resources
+ 创建logical device: 从一个==device group==(相同硬件属性)中选取subset physical devices==创建对应的logical device==
   + vkDeviceCreateInfo
   + 同时创建Queue

+ Device Use
   + 创建Queue
   + 创建Synchronization Construct
   + 管理内存
   + 管理CB
   + 管理graphics state
+ Lost Device: 
   + 原因：timeout, 电源与内存不足，implementation error
   + logical device lost时，不确保后续CB/device memory可靠，确保physical device可靠，可以再次创建logical device
   + 驱动implementation error时，可能导致physical device lost，不可创建logical device，甚至很可能导致app/os崩溃
   + host 仍需要==手动释放的内存(可达不可靠)==：
      + objects & devices
      + vkMapMemory绑定的host memory
      + external memory
   + async CB在 finite time 后返回VK_ERROR_DEVICE_LOST/VK_SUCCESS
   + semaphore行为由驱动保证
+ 销毁Device
   + vkDeviceWaitIdle -> vkDestroyDevice

##### Queue
+ queue family: queue执行相同op，通常一个physical device应该只有一个queue family

##### QueueFamily
+ vkDeviceCreateInfo.VkDeviceQueueCreateInfo
   + protected-capable queue（QUEUE_CREATE_PROTECTED内存独立）拥有唯一的queueFamilyIndex，与QueueFamilyProperties对应
+ QueueFamilyIndex与CommandPool对应
+ Image/Buffer需要指定可访问的QueueFamilies
+ 不同QueueFamily访问Resource，需要MemoryBarrier

##### Queue Priority（0.0~1.0）
高优先级分配更多时间片，不保证但有可能优先分配，仅在同一Device中
（Graphics > AsyncCompute）

##### Queue Submission
+ vkQueueSubmit(TargetQueue物理线程, batches, completionFence) 提交==到==物理线程后CB返回
+ 每个batch可以：
   + semaphore wait等待信号量开始执行
   + queue operation线程工作包
   + semaphore signal触发信号量（递减）
+ ==sparse memory binding绑定虚拟内存==
   + pagetable更新 --> 必须锁pagetable --> 所有相关CB被阻塞







