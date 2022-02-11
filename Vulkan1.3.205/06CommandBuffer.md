#### Command Buffers
+ vkCmdExecuteCommands
   + primary command buffer: 由cpu提交
   + secondary command buffer: 由primary command buffer提交
   + （为确保SCB不再被触发执行，）SCB invalid时，PCB必须也是invalid
   + PCB CB state 切换不影响SCB CB state
   + PCB CB state invalid时，切断与SCB CB state联系
   + PCB在recording/executable被再次record时，切换为Invalid
   + PCB与SCB的OcclusionQuery必须保持一致(?)
   + PCB不可以是ONE_TIME_SUBMIT/SIMULTANEOUS(?)
+ vulkan管线 state 通常对每个CB独立
   + CB record -> vulkan state undefined
   + 例外：render pass & subpass state
+ CB在queue中乱序执行，CB内外的任何memory dependency需要被显式指定
##### Command Buffer LifeCycle
+ vulkan CB state
      vkResetCommandBuffer/vkResetCommandPool
   + Initial: allocated, reset --  --> recording, freed
      ==vkBeginCommandBuffer(内部reset)==
   + Recording: 可以使用vkCmd*
      vkEndCommandBuffer
   + Executable: 可以被submit, reset, record to CB
      Queue Submission
   + pending: host 不可改， device 可达
      Device Complete Execution
   + CB_USAGE_ONE_TIME_SUBMIT ? invalid: executable
      + 修改或删除资源
   + Invalid: 只能被reset/freed

##### Command Pools
管理 CB，合理分配CB -> Queue，==执行所有CB state操作==
==CBPool external synchronized(线程不安全)==
+ vkCommandPoolCreateInfo.vkCommandPoolCreateFlags
   + TRANSIENT: 内存分配CB
   + RESET_CB: 允许BeginCmd隐式执行vkResetCommandBuffer（无RELEASE_RESOURCE_BIT）
   + PROTECTED：CB是protected CB
+ vkTrimCommandPool
   + 回收record后被reset的CB的内存
   + 很可能按bulk回收
   + 参考GC的压缩阶段，可能是2代GC后（不）放大0代内存空间（relief memory pressure）
+ vkResetCommandPool
   + ==该CBPool中的SCB会导致其他CB中的PCB invalid==
   + ==该CBPool中所有CB不可以是pending state==
   + vkCommandPoolResetFlag
      + RELEASE_RESOURCES: 是否回收所有内存资源

##### Command Buffer Allocation & Management
==CB externally synchronized(线程不安全)==
+ vkAllocateCommandBuffers
   + 任一CB失败，所有CB被释放
   + vkCommandBufferAllocateInfo.vkCommandBufferLevel
      + PRIMARY/SECONDARY
+ vkResetCommandBuffer
   + vkCommandBufferResetFlags
      + RELEASE_RESOURCE: 是否回收CB引用的内存资源
   + CB必须不在pending

##### Command Buffer Recording
+ vkBeginCommandBuffer
   + 对于SCB,vkCommandBufferBeginInfo.vkCommandBufferInheritanceInfo指向PCB
      + VkRenderPass&subpass(Index): 仅在RenderPass内时Validate
      + RenderPassTransform: 旋转CS.xy
      + VkFrameBuffer: SV_Target
      + VkQueryPipelineStatisticFlags: 可调用query的 pipeline statistic bit的最大集合（SCB可能沿用PCB的query-->统计数据$\leq$实际值）
         + IAVertice/Primitive
         + VS/PS/CSInvocation
         + GSInvocation/Primitives
         + TessCtrlShaderPatches/TessEvalShaderInvocation
         + ClipInvocation/Primitives
      + VkCommandBufferInheritanceConditionalRenderingInfoEXT：是否可以继承ConditionalRendering
      + vkViewportInfo：是否可以继承Viewport->==同一PSO的所有SCB都设置/都不设置Viewport==
      + 以上所有资源必须来自同一Device
   + CB必须不在recording/pending 
   + vkCommandBufferBeginInfo.vkCommandBufferUsageFlag
      + ONE_TIME_SUBMIT: execute->reset->record
      + RENDER_PASS_CONTINUE: 这是一个SCB inside RenderPass
      + ==SIMULTANEOUS: 可以在pending时被再次submit && 可以被record到多个PCB中==
         + ==增加cpu并发量，增加gpu内存占用（CB内的内容必须被复制后执行）==
+ vkCmdExecuteCommands
   + 数组内==第i个SCB继承第i-1个SCB设置好的状态==
   + 绑定PSO会使“继承的状态”失效
   + undefined(如viewport深度与minmax不对应)，状态不被继承
   + 不同PCB的SCB不继承状态
+ vkEndCommandBuffer
   + error时CB必须被reset

##### Command Buffer Submission ==(Draw/Call)==
尽可能减少vkQueueSubmit/vkQueueSubmit2KHR
提交后CB executable -> queue pending -> CBs complete -> queue executable/(存在一个ONE_TIME_SUBMIT)queue invalid
==执行失败时，驱动确保内存资源&锁不受影响/驱动返回DEVICE_LOST==
+ vkQueueSubmit2KHR
   + vkQueue: 一个提交线程
   + vkFence: 在所有CB执行完成后触发,必须未触发，必须不受其他未完成的queue影响
   + pSubmits: batch数组
      + ==PCB&SCB必须在pending(仅SIMULTANEOUS)/executable==
      + ==CB必须由同一个queue family的CBPool分配==
         + 或者QueueFamily(Ownership)TransferRelease->Acquire
+ vkEvent
   + CB引用的vkEvent不可被其他queue的pending CB引用
   + vkCmdWaitEvent不能在RenderPass内，vkSetEvent必须在此前被触发
+ semaphore & stageMask 管线(shader)状态
   + CB引用的semaphore必须只被触发一次(?), 其WaitInfo/SignalInfo的stagemask必须被queue的pipeline stages支持，semaphoreWait上只能有一个queue
   + WaiteInfo的semaphore必须在此前有CB Signal
   + timeline semaphore必须 #Wait<#Signal< MaxTimelineSemaphoreDiff
   + semaphore必需持有fence payload
   + semaphore可能借用D3D11 keyed mutex(?)

##### Queue Forward Progress
app必须确保queueSubmit内的CB都可以被执行（不被semaphore阻塞）
vulkan不保证驱动有timeout

##### SCB Execution
如果两个executable/recording PCB 录入 同一个 非SIMULTANEOUS的SCB，PCB --> Invalid
vkCmdBenginRenderPass必须设置SUBPASS_CONTENTS_SECONDARY_CB
+ RenderPass内的SCB
   + 必须设置CB_USAGE_RENDERPASS_CONTINUE
   + 必须有相同的PassInfo::transform,renderArea
   + 必须有大致相同的QueryStatistics
##### CommandBufferDeviceMask
deviceIndex 决定此后的所有CB在哪个physical device上执行
scissor/viewport在不同physical device上可以不同
其他vulkan state在不同physical device共享
actionCmd前一个stateSettingCmd必须包含actionCmd的所有deviceIndex
Begin/EndRenderPass&&Subpass的deviceMask均由PassInfo指定




   
   
      