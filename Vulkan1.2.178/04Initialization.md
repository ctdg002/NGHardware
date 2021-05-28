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




