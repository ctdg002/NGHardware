#### Fundamentals
##### Host-cpu & Device-gpu 必要环境
8,16,32,64bit int
32,64bit float(TODO:精度要求)
endian与硬件相同
*datatype考虑与host&device兼容


##### Architecture
+ Vulkan Layered API
   + core Vulkan Layer
   + Debug Layer
   + Validation Layer：assert API调用

##### Execution Model（硬件多线程）
###### GPU多线程 -- devices进程 -- queue families线程池队列 -- queues线程
+ queue families: 相似性质、互相兼容的线程，承接某一种work
    + eg: video encode/decode，graphics,compute,transfer, sparse memory management

###### Memory heap -- typed memory area
所有类型均为device visible
   + device local 
   + device local, host visible
   + host local, host visible  
   

###### CommandBuffer 
可反复提交，多线程创建、执行，在一个gpu线程上同步执行，与cpu异步执行
+ queue submission in batches
    + vkQueueSubmit/vkQueueBindSparce 
    + 顺序受[order constraints同步时机](https://www.khronos.org/registry/vulkan/specs/1.2-khr-extensions/html/vkspec.html#synchronization)影响
       + 开始时等待一个semaphore，结束时trigger一群semaphore/fence =》写转读
       + 所有顺序依赖仅依靠semaphore/fence
       + 提交后，current state is reset on each boundary
+ ==不同queue(承载CB)之间乱序执行==
    + 只要Action CB的执行状态一致，一起运行
    + State Setting CB更新current state，不可一起运行
    + synchronization CB引入explicit依赖，trigger action CB
       

##### Vulkan Object
所有对象“first parameter”/可访问性和作用域为VkDevice
###### Object -- Handle
+ dispatchable: UniquePtr，指向实例对象
+ non-dispatchable: 虚基类
   + 虚基类由int64成员伪装而成时：带info的假handle，非UniquePtr可能被多个“持有者”复用，不影响“持有者”构造析构
+ external: 必须被import&export
###### Object LifeCycle
+ vkCreate* & vkDestroy*
   + 无pool => 低频，更慢
+ vkAllocate* & vkFree*
   + 从pool/memory heap中获取 => 高频，更快
+ ownership: app负责析构，析构先子后父

+ vkObjects父子关系
vkInstance 
   + vkDevice可析构
      + vkQueue在vkDevice卸载时析构
      + vkFence,vkSemaphore,vkCB,vkCBPool在线程返回时析构
      + vkXxxPool析构时自动析构vkXxx
      + vkRenderPass：
           + vkPipelineLayout: 所有使用它的CB在recording state
           + vkDescriptorSetLayout: 所有CB不再vkUpdateDescriptorSets
      + 其他：CB完成析构，CB pending state视为未完成，所有线程idle时可析构其他obj
   + vkPhysicalDevice不可析构

##### API Syntax
###### Application Binary Interface 跨平台能力
vk_platform.h as Shared Library
implementation: Application Binary Interface via c
+ 数据结构：size, align, layout
+ 函数calling convention：
   + VKAPI_ATTR/VKAPI_CALL 返参
   + VKAPI_PTR * 指针函数返参
+ Symbol&Naming
   + ==由vk开头的符号是ABI保留关键字，app不可使用==

+ 【？】获取更高版本feature方式
   + vkGetInstanceProcAddr/vkGetDeviceProcAddr

###### Command Syntax & Duration
基本数据结构：c99,stdint.h
uint32_t VkBool32 -- VK_TRUE,VK_FALSE
uint64_t VkDeviceSize -- 内存大小&offset
uint64_t VkDeviceAddress -- 指针大小[buffer address value?]  
构造函数：
vkCreate(VkCreateInfo, pAllocator)/vkDestroy
vkAllocate(vkAllocateInfo)/vkFree,池化obj沿用Pool的allocator
RetrieveResult：vkGet/vkEnumerate
##### Queues
###### Threading Behavior
默认多线程，传统mutex需要通过pthread library等第三方库构建
被定义为externally synchronized的vkObject：需要线程安全的SimpleStoreUpdate-> app为所有相关CB设置内部memory barrier，确保CB不同时执行
   + 对线程的操作：Queue，Fence，Semaphore
   + 事件触发：Event
   + 特殊资源：CB本身，swapchain
   + 将被Bind/Map的资源：Memory/Buffer
   + 将被CBDestroy的资源：Sampler，ShaderModule，Pipeline，CBPool, DescriptorPool，Device
暂时移交给vkCmdBuffer的application-owned memory(成为non-const parameter)：直到deferred CB完成
含有externally synchronized的vkObject的数组：每个element分别保证线程安全
   + semaphore
   + buffer/image/discriptor/surface
   + swapchain
implicit external synchronized的vkObject：不直接作为parameter传入，但是会受到对特殊parameter执行相关CB操作的影响
   + 对vkDevice析构时，影响vkQueue
   + 分配CB时，影响CBPool
immutable(其他所有parameter)：不需外部sync，不可destroy，可能内部sync


##### Pipeline Configuration & Numeric Representation
###### Error
可能程序终止，不会影响OS，遵循OS约定（进程内存不越界，初始值，use-after-free不引起其他进程内存泄漏）

###### Valid Usage Conditions @ Validation Layer
仅编译时可知，函数调用时对传参类型检测，所有State提交完成时检测DrawState
###### Implicit Valid Usage
+ Object Handle：成功创建，未释放，成员与passBy合法
   + VK_NULL_HANDLE/NULL可以被传入vkDestroy/vkFree
+ pointer：类型安全、align合法的内存
+ string：char[]，以NULL结尾
+ Enum：enumerant有定义，非_MAX_ENUM(c 32b-Enum标志)，非为扩展预留的reserved value
   + 为extension使用switch的default
+ Flags：
   + uint32_t == vkFlags 
   + "enumType": Vk*Flags
   + "enumerant": Vk*FlagBits，0x80000000
   + ==会存在未定义的bit flag位==
   + ==仅30~0bit合法==
+ vk内置Structure：
   + VkStructureType sType成员，表示Structure为XxxInfo，与结构名对应
   + void *pNext成员，指向下一个ExtendingStructure/NULL    
      + Structure Pointer Chain：不重复，平台支持
   + 基本结构体：
       + VkBaseInStructure迭代cpu发送到gpu信息
       + VkBaseOutStructure迭代gpu返回cpu信息
   + 有nested结构体
+ Extension：
   + Instance Level feature
      + 在vkEnumerateInstanceExtensionProperties中
      + 被vkInstanceCreateInfo支持
   + Physical-device-level需要被[ExtendingPhysicalDeviceCoreFunctionality](https://www.khronos.org/registry/vulkan/specs/1.2-khr-extensions/html/vkspec.html#initialization-phys-dev-extensions)支持
   + Device Functionality
      + vkEnumerateDeviceExtensionProperties
      + VkDeviceCreateInfo
+ Newer Core Version：
   + Instance Level
      + vkEnumerateInstanceVersion
      + VkApplicationInfo::apiVersion
   + (Physical) Device Level
      + VkPhysicalDeviceProperties::apiVersion
      + VkApplicationInfo::apiVersion
###### State and State Query: Return Code 返回码
+ VkResult-Success Code
   + VK_NOT_READY：fence/query未完成
   + VK_THREAD_DONE_KHR：deferred operation未完成，但没有剩余线程工作
   + VK_SUBOPTIMAL_KHR：swapchain不再符合surface描述，但依然可以工作
   + VK_INCOMPLETE：返参数组还不足够
+ VkResult-Error Code
   + VK_ERROR_*
   + VK_ERROR_INCOMPATIBLE_DISPLAY_KHR [?]backbuffer不同
   + VK_ERROR_INVALID_OPAQUE_CAPTURE_ADDRESS [?]Buffer/ShaderGroup内存不可达
   + VK_ERROR_UNKNOWN：application/implementation出错 -> 使用最新Validation Layer验证app
+ Errror性质：
   + RuntimeError时，所有output parameter UB
   + VK_ERROR_OUT_OF_*_MEMORY不会导致此前的Object UB
   + 一些性能关键的CB可能延迟error report

###### Numeric Representation & Computation
shader的range & precision 见[SPIR-V](https://www.khronos.org/registry/vulkan/specs/1.2-khr-extensions/html/vkspec.html#spirvenv-precision-operation)

非shader的range & precision 见[data format](https://www.khronos.org/registry/vulkan/specs/1.2-khr-extensions/html/vkspec.html#data-format)
https://www.khronos.org/registry/DataFormat/specs/1.3/dataformat.1.3.html
+ 数据格式
   + 16b = 1 sign + 5 exponent + 10 mantissa
   + u11b = 5 exponent + 6 mantissa
   + u10b = 5 exponent + 5 mantissa
+ 浮点数range & precision不合法不导致崩溃
   + 浮点数magnitude >= $2^{32}$
   + unsign负数 $\Rightarrow$ 0
   + 超过magnitude $\Rightarrow$ Inf或最大数
###### Fixed-Point Int $\Leftrightarrow$ normalized Floating Point Conversion
signed fixed-point integer: signed two's complement $\in[-2^{b-1}+1,2^{b-1}-1]$
floating point : $f = max(\frac{c}{2^{b-1}-1}, -1.0) \in [-1,1]$
注意$-2^{b-1}$在int中合法，在fp中会被clamp


##### Common Object Types
###### Offsets
buffer中的uint32 pixelPos
###### Extents
buffer中的uint32 长方形区域(width,height,depth)
###### Rectangles = Offset + Extent
###### StructureType
见VkStructureType




