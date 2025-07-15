# SGLang PD 开源的实现解析

## Scheduler
scheduler是sglang主要处理每一个gpu推理任务的部分，和非pd分离形式下的sglang scheduler是一致的，开启pd分离之后，会额外构建出pd分离过程中需要的数据结构，即下文中的几个队列。

## Prefill Server 
对req进行管理的队列如下，req依次传递

### Bootstrap Queue
构建这个队列的时候，会创建一个kv mananger负责kv cache的传输。
- 首先把请求放入到这个队列中，为每一个请求创建一个kv sender
- 存储正在进行握手的sender
- 检查pd之间握手是否成功，d节点握手成功之后会通知p节点
- 握手成功之后，放到下一个环节队列中
 

### Waiting Queue
- 执行推理计算
- 把计算完成的req放到下一队列，进行kv cache的传输

### Infight Queue
- 调用kv sender开启非阻塞多线程的kv cache传输

### 执行流程
- 每一个req先放入bootstrap queue中，这个时候会给每一个req构建一个kv sender，sender负责之后kv cache计算完的传输api的调用。
- 等待d节点通知，可以开始进行kv cache的计算
- 计算完成之后调用sender send() api只需要传递kv cache的page indices即可
- prefill manager会有一个线程池，等待进行kv cache的传输任务

## Decode Server 

### Prealloc Queue 
- 给每一个req创建一个kv receiver
- 每一个receiver根据req的bootstrap server信息（load balancer设定的），找到对应的bootstrap server，得到p节点的tp参数和信息，此时握手成功，握手成功之后会通知p节点。
- 握手成功之后放入到下一队列，开始等待接受kv cache
- 构建这个队列的时候，会创建一个kv mananger负责kv cache的接收

### Transfer Queue:
- 等待kv cache的req，nixl和mooncake对于kv cache的成功传输，通知方式不同
- 通知接受到kv cache之后，放入下一队列

### Waiting Queue:
- 进行decode阶段的推理计算
- 推理完成，返回完整的req推理结果

### 执行流程
- 每一个req先放入prealloc queue中，会给每一个req构建一个kv receiver，receiver会从bootstrap server里订阅获取对应p节点的信息，然后根据获得的prefill
 tp rank的信息，通知prefill节点握手成功.
- 当decode节点给req分配资源成功之后，会通知p节点，可以开始进行kv cache的传输。

## Bootstrap Server 控制节点

### 构建和启动
启动lb（load balancer）的时候可以提供prefill host ip和bootstrap port参数，这两个参数构成了bootstrap server的地址。
只有node rank 0 sglang server会启动一个TokenizerManger，TokenizerManger会根据是否开启了pd，启动bootstrap server。

### 使用
负责记录p节点信息，等待d节点访问。
在req经过load balancer之后，会被分配bootstrap server的设置，之后req发送给所选定的p节点，d节点。所以pd节点后续知道该如何访问哪一个bootstrap server获得信息。
保存的控制信息，每一个prefill tp rank的具体地址。

## kv cache传输使用到的类
kv manager， sender， receiver三个传输类由具体的cache transfer engine来实现，开源的由mooncake和nixl两个后端实现，同时对于基本的pd握手和调度功能，有共同的公共API接口，例如kv sender send()发送cache， poll()检查状态，receiver init()通知握手成功。
### kv poll

控制信息，负责记录一个kv传输事件的状态

### kv manager
Prefill manager （位于Bootstrap Queue）
manager会向bootstrap server注册所在的prefill rank的信息，信息如下，成功注册之后bootstrap server上就有了这个prefill tp rank的信息。

manager会等待d节点分配资源的信息，即是否能够握手成功，同时等待d节点的控制信息（TransferInfo），其中包括kv cache传输目标d节点所对应的kv indices信息，这个信息之后会提供给rdma 传输api进行cache的传输。

manager维护一个dict，记录所有kv传输req的其对应的传输控制信息

bootstrap room即为每一个req的随机UID。TransferInfo即为对应d节点的传输目标信息。
Decode manager （位于Prealloc Queue）
等待p节点kv cache传输成功的通知，

维护一个表，映射bootstrap addr (host:port) -> prefill节点tp参数的信息。
每一个req到达decode节点并且添加到队列中之后会拥有一个kv receiver，通过bootstrap addr获得对应p节点的参数信息。
开启线程进行与bootstrap server的通信心跳检测。
sender
维护kv cache传输的状态，调用kv manager进行实际的kv cache传输。
公共send()的参数都是kv cache所对应的page id，具体的传输，由后端的transfer engine根据req参数和传入的page id执行。
receiver
从bootstrap server参数(load balancer设定的)，获得对应p节点的tp参数，更新decode manager的表格信息。
获得p的tp信息之后，通过查询bootstrap server获得每个p tp rank的地址信息，得到所有需要交互的p节点信息。
通知p节点，握手成功，同时传递d节点的地址信息给p节点。
pd握手流程
每个req receiver构建的时候会通过bootstrap server得到所有相对应的p节点地址和信息。
当每个req放入Prealloc Queue之后，scheduler event loop会不停检查这个队列，尝试分配资源给req，成功分配资源的req会通过receiver init()函数通知其对应的p节点，握手成功，此时表示pd握手成功，同时pd节点互相知道对方地址。
kv cache传输的实现
每一个req有其对应的sender，req cache计算完成之后，调用自己的sender执行kv cache的公共api send()，参数为kv cache对应的page id，后续nixl和mooncake的传输实现会有不同。形式上都是sender调用对应的kv manager把kv cache transfer请求放入队列中，等待发送。
mooncake通过p节点传输完成之后，通知d节点kv cache传输完成。
nixl通过d节点poll检查kv cache是否传输完成。
地址和token索引是如何组织的:
req_to_token_pool:  req id -> kv indices
token_to_kv_pool:  kv indices -> kc cache tensor

每一个req维护start_send_index和end_index，每一次send_chunk，会传输[start:end]范围的kv indices，根据page size映射到对应的page id，传递给下层数据传输引擎。
