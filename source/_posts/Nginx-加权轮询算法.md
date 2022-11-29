---
title: Nginx 加权轮询算法
top_img: https://s2.loli.net/2022/07/20/tOoYKRE7fMVdb6x.png
cover: https://www.apple.com.cn/newsroom/images/product/app-store/Apple_Tech-Talks-2021_10202021_big.jpg.large_2x.jpg
date: 2022-07-24 21:47:53
tags:
    - 网络
    - 算法
    - golang
categories: 
           - [算法]
           - [网络]
---
# 轮询 （round-robin）
>简单的说一下轮询
1. nginx 中的配置
``` nginx
upstream cluster {

server 192.168.0.14;
server 192.168.0.15;
}
location / {

proxy_set_header X-Real-IP $remote_addr; //返回真实IP
proxy_pass http://cluster; //代理指向cluster
}
```
2. 简单介绍
#### 轮询 作为负载均衡中较为基础的算法，他的实现不需要配置额外的参数。简单理解：配置文件中一共配置了 N 台服务器，轮询 算法会遍历服务的节点列表，并按照节点顺序每轮选择一台服务器处理请求，当所有节点遍历一遍后，重新开始.

3. 算法中不难看出，每台服务器处理请求的数量基本持平，按照请求时间逐一分配，因此只能适用于集群服务器性能相近的情况，平均分配让每台服务器承载量基本持平。但是如果集群服务器性能参差不齐，这样的算法会导致资源分配不合理，造成部分请求阻塞，部分服务器资源浪费。为了解决上述问题，我们将 轮询 算法升级了，引入了 加权轮询 算法，让集群中性能差异较大的服务器也能合理分配资源。达到资源尽量最大化合理利用

4. 实现 
``` golang
type RoundRobinBalance struct {

curIndex int
rss []string
}
/** * @Author: yang * @Description：添加服务 * @Date: 2022/7/25 8:36 */
func (r *RoundRobinBalance) Add (params ...string) error{

if len(params) == 0 {

return errors.New("params len 1 at least")
}
addr := params[0]
r.rss = append(r.rss, addr)
return nil
}
/** * @Author: yang * @Description：轮询获取服务 * @Date: 2022/7/25 8:36 */
func (r *RoundRobinBalance) Next () string {

if len(r.rss) == 0 {

return ""
}
lens := len(r.rss)
if r.curIndex >= lens {

r.curIndex = 0
}
curAdd := r.rss[r.curIndex ]
r.curIndex = (r.curIndex + 1) % lens
return curAdd
}
```
5. 测试

简单调用下方法查看结果
``` golang
/**
* @Author: yang
* @Description：测试
* @Date: 2022/7/25 8:36
*/
func main(){
rb := new(RoundRobinBalance)
rb.Add("127.0.0.1:80")
rb.Add("127.0.0.1:81")
rb.Add("127.0.0.1:82")
rb.Add("127.0.0.1:83")
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
}
```

``` shell
go run main.go
127.0.0.1:80
127.0.0.1:81
127.0.0.1:82
127.0.0.1:83
127.0.0.1:80
127.0.0.1:81
``` 
# 二，Nginx 负载均衡的加权轮询 （weighted-round-robin）

### nginx 配置
``` nginx
http {

upstream cluster {

server 192.168.1.2 weight=5;
server 192.168.1.3 weight=3;
server 192.168.1.4 weight=1;
}
location / {

proxy_set_header X-Real-IP $remote_addr; //返回真实IP
proxy_pass http://cluster; //代理指向cluster
}
```
### 加权算法简介-特点
不同的服务器的配置，部署的应用数量，网络状况等都会导致服务器处理能力会不一样，所以简单的 轮询 算法将不再适用，而引入 了加权轮询 算法：根据服务器不同的处理能力，给每个服务器分配不同的权值，根据不同的权值将不同的服务器分配到对应的服务器上；

>请求数量较大时，每个服务处理请求的数量之比会趋向于权重之比。

### 算法说明

在 Nginx加权轮询算法 中，每个节点都有3个权重的变量

+ `Weight` : 配置的权重，根据配置文件初始化每个服务器节点的权重

+ `currentWeight` : 节点的当前权重，初始化时是配置的权重，随后会一直变更

+ `effectiveWeight` : 有效的权重，初始值为 weight ，通讯过程中发现节点异常，则 -1 ，之后再次选择本节点，调用成功一次则 +1 ，直到恢复到 weight。这个参数可以用于做降权。或者说是你的设置的权限修正。。

Nginx加权轮询算法 的逻辑实现

1. 轮询所有节点，计算当前状态下所有的节点的 effectiveWeight 之和 作为 totalWeight；

2. 更新每个节点的 currentWeight ， currentWeight = currentWeight + effectiveWeight; 选出所有节点 currentWeight 中最大的一个节点作为选中节点；

3. 选择中的节点再次更新 currentWeight, currentWeight = currentWeight - totalWeight；

### 简单举例

>注意：实现中不考虑健康检查，即所有的节点都是100%可用的，所以 effectiveWeight 等于 weight

假设：现在有3个节点 {A, B, C} 分别权重为：{4, 2, 1}；请求7次
| 第N次请求 | 请求前 currentWeight                    | 选中的节点   | 请求后 currentWeight                    |
|-------|--------------------------------------|---------|--------------------------------------|
| 1     | \[serverA=4, serverB=2, serverC=1\]  | serverA | \[serverA=1, serverB=4, serverC=2\]  |
| 2     | \[serverA=1, serverB=4, serverC=2\]  | serverB | \[serverA=5, serverB=-1, serverC=3\] |
| 3     | \[serverA=5, serverB=-1, serverC=3\] | serverA | \[serverA=2, serverB=1, serverC=4\]  |
| 4     | \[serverA=2, serverB=1, serverC=4\]  | serverA | \[serverA=-1, serverB=3, serverC=5\] |
| 5     | \[serverA=-1, serverB=3, serverC=5\] | serverA | \[serverA=3, serverB=5, serverC=-1\] |
| 6     | \[serverA=-1, serverB=3, serverC=5\] | serverA | \[serverA=3, serverB=5, serverC=-1\] |
| 7     | \[serverA=0, serverB=7, serverC=0\]  | serverB | \[serverA=4, serverB=2, serverC=1\]  |
>totaoWeight = 4 + 2 + 1 = 7
>第一次请求：serverA = 4 + 4 = 8 ， serverB = 2 + 2 = 4， serverC = 1 + 1 = 2；最大的是 serverA ；所以选择 serverA ；然后serverA = 8 - 7 = 1；最后得出：serverA=1, serverB=4, serverC=2
>第二次请求：serverA = 1 + 4 = 5；serverB = 4 + 2 = 6 ；serverC = 2 + 1 = 3；最大的是 serverB ；所以选择 serverB ；然后 serverB = 6 - 7 = -1 ；最后得出：serverA=5, serverB=-1, serverC=3
>以此类推……

### 代码实现
以golang实现下上面的逻辑：
``` golang
type WeightRoundRobinBalance struct {

curIndex int
rss []*WeightNode
}
type WeightNode struct {

weight int // 配置的权重，即在配置文件或初始化时约定好的每个节点的权重 
currentWeight int //节点当前权重，会一直变化 
effectiveWeight int //有效权重，初始值为weight, 通讯过程中发现节点异常，则-1 ，之后再次选取本节点，调用成功一次则+1，直达恢复到weight 。 用于健康检查，处理异常节点，降低其权重。 
addr string // 服务器addr 
}
/** * @Author: yang * @Description：添加服务 * @Date: 2022/7/25 8:36 */
func (r *WeightRoundRobinBalance) Add (params ...string) error{

if len(params) != 2{

return errors.New("params len need 2")
}
// @Todo 获取值 
addr := params[0]
parInt, err := strconv.ParseInt(params[1], 10, 64)
if err != nil {

return err
}
node := &WeightNode{

weight: int(parInt),
effectiveWeight: int(parInt), // 初始化時有效权重 = 配置权重值 
currentWeight: int(parInt), // 初始化時当前权重 = 配置权重值 
addr: addr,
}
r.rss = append(r.rss, node)
return nil
}
/** * @Author: yang * @Description：轮询获取服务 * @Date: 2022/7/25 8:36 */
func (r *WeightRoundRobinBalance) Next () string {

// @Todo 没有服务 
if len(r.rss) == 0 {

return ""
}
totalWeight := 0
var maxWeightNode *WeightNode
for key , node := range r.rss {

// @Todo 计算当前状态下所有节点的effectiveWeight之和totalWeight 
totalWeight += node.effectiveWeight
// @Todo 计算currentWeight 
node.currentWeight += node.effectiveWeight
// @Todo 寻找权重最大的 
if maxWeightNode == nil || maxWeightNode.currentWeight < node.currentWeight {

maxWeightNode = node
r.curIndex = key
}
}
// @Todo 更新选中节点的currentWeight 
maxWeightNode.currentWeight -= totalWeight
// @Todo 返回addr 
return maxWeightNode.addr
}
```

### 测试验证
``` golang
/** * @Author: yang * @Description：测试 * @Date: 2022/7/25 8:36 */
func main(){

rb := new(WeightRoundRobinBalance)
rb.Add("127.0.0.1:80", "4")
rb.Add("127.0.0.1:81", "2")
rb.Add("127.0.0.1:82", "1")
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
fmt.Println(rb.Next())
}
```
执行看下结果：
``` shell
run main.go
127.0.0.1:80
127.0.0.1:81
127.0.0.1:80
127.0.0.1:80
127.0.0.1:82
127.0.0.1:80
127.0.0.1:81
```
