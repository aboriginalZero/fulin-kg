申请 grant，用的还是 req num，但应该看 req num 和 bytes 都应该考虑到，

ifc 给 ifm 的 req 里，或许需要再加上几个字段。

* 如果 ifm 给各个 ifc 还是按照目前只看 iops 颁发 token 的话。考虑场景 ifc1 上有 99 个 recover，1 个 sink，ifc2 上有 1 个 recover，99 个 sink。ifm 第一个轮给到二者的 token 数量是等大的。但是 recover 是大 io，要消耗的 token 数量多，那么完成的 io 个数就少，下一轮获取 token 的时候，就变少了，长期下去，就会越来越少。（这个问题应该还好，下一轮获取 token 传递的 last_used_num 虽然变小了，但是 req_num 还是大的）
* 如果只看 bytes，也会有问题，因为 bytes 大的拿到的 token 就多，那么低 io 的，完成会慢一些，对大 IO 有利，但大 IO 不一定优先级就高，比如 migrate 就低。

搞个 waiting_bytes



ifm 中的 token 消费方式会有点问题。

1. 在有多种 internal io 共存的情况下，按目前这种 4 : 2 : 1 分 token，可能会存在浪费情况，不会是控制在一定范围内的。比如有 63 个 token 给到了 recover，但 recover replica 都是 64 个的，所以 recover io 也 resume 不了，但其实可以给到 sink io 使用的
2. 另外，假设 ifc 只有 sink io 要放行，但从 ifm 得到的 token 数量有限，若 token 数不能被整除，可能出现先到的 256k sink io 由于拿不到 64 个 token 被阻塞，后到的 4k sink io 一直在放行。
3. 对于 migrate / recover，如果总是 256 KiB io，而剩余 token 又小于 64 个的话，就会因为都消费不了而遍历 waiting_io_list_[i] 的全部元素，时间开销会更大些。

造成差异的原因是目前的配额是按 4k 粒度的 token 的 4 : 2 : 1 的方式分的，还应该考虑 bytes。这个问题在限速不低的情况下，应该还好，且最终总能收敛。



时刻 t - 1 的信息：

1. granted num
2. avail bucket level

时刻 t 的信息：

1. last used num
2. over used num
3. req num
4. avail bucket level

求时刻 t 的 granted num

能够知道 t - 1 时刻颁发了多少 token，flow ctrl 消费了多少，

上一时刻的 granted num - 这个时刻的 (last used num + over used num) 表明这个时段内 flow ctrl 消费了多少 token，即有多少 io 下发了，剩下的 token 回收并在下一次使用。

这个时刻的 avail bucket level - 上一时刻的 avail bucket level ，如果是负数，表明可用额度变少了，时刻 t 给出的 granted num 应该要变小；如果是正数，



ifm 计算给每个 ifc 的 current granted num 时，认为的所有 chunk 能用的 avail token 之和包含 2 部分：

1. 这段时间内的 leak bytes
2. 每个 ifc 的 prevent granted num - (current last used num + current over used num)，即上一轮未被消费的部分回收以供下一轮使用；

这种方式需要考虑：

1. 节点异常等不能提供 last used num 等信息时，需要及时回收之前发给他的 token；
2. 收集到所有 ifc 的请求后再计算 avail tokens，目前已有这个逻辑；
3. 需要考虑回收的是 2 轮前的。

avail bucket level 的更新频率远低于 100ms，相比 local io stats 记录的更为精准。另外，这种方式不会有 inflight io 带来的误差，因为：

1. 若新 io 下发且处于 inflight，体现在 current last used num ++，不影响 bucket level；
2. 若新的 io 下发且完成，体现在 current last used num ++，bucket level --；
3. 不关心旧的 inflight io 是否完成；



不可以，考虑 io throttle 每 10 ms 减一次 bucket level，ifm 每 100ms 管 io throttle 要一次 avail bucket level，那么正常情况下能拿到 10 个可用 bucket 空间，这轮 granted num = 10， ifc 据此发了 10 个 io，但都在 inflight，还没执行到 level ++，那么 ifm 下一个 100ms 去找 io throttle 要到的 avail bucket level 是 10 + 10 = 20 个，这轮的 granted num = 20，ifc 据此发送了 20 个 io。假设所有 io 都在 inflight，那 granted num 就会是 10，20，30，40 的递加，而实际上应该是 10，10，10，10。



引入 internal flow ctrl 的目的是为了下发到 internal io throttle 的时候不会被卡住。

reposition 动态并发度限制，对于 perf，默认限制是 32 个，最多会有 128 * 64（chunk 个数）= 8096 个 perf block flight io 打到同一个 local io handler 上，256 KiB/s * 8096 = 2 GiB/s。



之前是设想 internal io throttle 放在 io 开始前，但如果由于需要关注 ELSMNotAllocData 以及记录真实的 io bytes（而非固定的 block size）而把 internal io throttle 放在 io 完成后，他是否也可以根据 io done 的信息来限制？暂时不考虑改变 io throttle 机制。

local io handler 里难以保证是否会 yield，如果确保不会，可以添加。