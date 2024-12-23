* 即使是消息推送，gangzi 都做了 4 年，即使是冷启动一件事，douyin 都投入了 20 多人的团队在优化，对于自己手上的工作，要尽快从点切换到面的角度解决问题，从全链路来看，每个环节都是草台班子，都有可以优化的地方。fanggang
* 保持一个 patch 只做一件事，当解决问题 A 的同时发现 B 有类似的错误，首先确认 B 是真的会有错误还是只是别人写的代码自己看的不习惯，如果是真有错误，确认下影响范围，确认要修改的话，再考虑另提一个 patch，对于提交的每个 patch，要考虑引入风险的范围。zongjie
* 当被高优先级任务中断时，即使中间搭环境要等时间，也应该在这个时间里想好哪些路径会导致这个问题并列出来，这样等环境弄好后能够快速验证猜想，避免频繁地上下文切换才能保证效率最大化。yutian
* review 优先级在个人研发工作前头，在 release 阶段做 bugfix 改动范围应该尽可能小，一些语法类的非必要改动就不要动。yutian
* 优先做自己的 patch，把 review 别人 patch 的任务放在最后，关注自己能经得起时间检验的产出。yiwu
* 要让花出去的时间有对应的产出，只是 review 别人的代码、调查问题但最终不是自己提 patch 解决问题的话，是很吃亏的。yiwu
* 只要业务能尽快上线，快速地用粗糙的技术 a 胜过慢吞吞地用自认为优雅的技术 b，技术服务于业务，用于实现商业价值。yuhang
* 主动将自己的工作宣传到其他成员/组别，并用一部分精力关注他们的工作。yuhang
* 当有 project a 和 b，只在上班时间内做重要的 a，无需花自己的下班时间做相对不重要的 b。yuhang
* 做 leader 认为重要的事，对项目进度有帮忙的事，而不是自己认为重要的事，优先做即使重复枯燥、但更影响绩效评估的工作。yuhang
* 忙工出细活和快速出活并不互斥，应该修炼能在二者快速切换的能力。yutian
* 不做 delay project 最后一个人，先把主体功能交上去后再做 bugfix，做一个功能，要让代码改动尽可能小。yutian
* 即使是策略类代码，也有可以创新的地方，yutian 接手的时候是没有局部化策略的，那我目前只是在已有策略上演进。yiwu
* 写策略类代码宁愿笨一点，把 if else 多种情况罗列全，也不要图方便整合过多的逻辑。代码简单是面向人类逻辑的简单，不是代码数量少。yiwu
* 所有的策略讨论的时候，可以尽量展开相对多的可能性，但是落地的时候一定要收敛。适当的额控制粗糙程度，即便是复杂和精细的策略落地的时候也要用简单的方式呈现在文档和代码上。重点是让人理解。yutian
* 考虑兼容性，reroute_version_update 的增加和 rsync 的删除都要考虑到系统可能还有旧版本在运行。yutian
* 一个知识点比较难，别人讲过几次还是没理解，那么再次请教的时候，先说出自己会的东西，然后用新的角度提问自己不会的东西。yutian
* 一开始不用做大而全的设计，因为在没经验的时候也做不好。为了满足业务，即使代码冗余也没事。在面对性能和兼容性的时候，不一定说得一方面有很大的牺牲才能做好另一方面。中间一定有更好的应对方法，试着去抽象、能够做到好的应对当前性能需求的同时，也留有后续迭代的余地。wangsai
* 先猜想再设计，比如网络相关，先理清网络拓扑，然后抓包验证猜想。不可能 100% 搞清楚再开始，都是摸着石头过河，先提出假设，再用实验验证假设。yutian





Many git tutorials focus on a set of commands and instructions to “get you up to speed” in git, without addressing the underlying concept of “how git works”. While the commands are important, I feel it’s more important for you to understand what’s going on behind the scenes. (Full reference material is included in the appendix at the end of this document. Most diagrams are taken from Pro Git.)

At a high level, git can be thought of as a database/filesystem/backing store that remembers a truckload of details about your code base. This information is called the git repository, and contains three types of content: blobs, trees and commits.

necessary --> essential

important --> significant

at the same time --> simultaneously

the alternate approach，一种可选的方式

think of A in terms of B，从 B 的角度来考虑 A