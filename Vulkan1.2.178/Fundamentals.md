#### Fundamentals
##### Host-cpu & Device-gpu 必要环境
8,16,32,64bit int
32,64bit float(TODO:精度要求)
endian与硬件相同
*datatype考虑与host&device兼容


##### Architecture

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


##### Application Binary Interface

##### API Syntax

##### Queues

##### Pipeline Configuration

##### Numeric Representation


##### State and State Query


##### Objects 


##### Shaders