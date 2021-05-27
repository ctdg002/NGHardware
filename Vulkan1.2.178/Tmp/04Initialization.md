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




