* 先猜想再设计，比如网络相关，先理清网络拓扑，然后抓包验证猜想。不可能 100% 搞清楚再开始，都是摸着石头过河，先提出假设，再用实验验证假设。yutian
* 一开始不用做大而全的设计，因为在没经验的时候也做不好。为了满足业务，即使代码冗余也没事。在面对性能和兼容性的时候，不一定说得一方面有很大的牺牲才能做好另一方面。中间一定有更好的应对方法，试着去抽象、能够做到好的应对当前性能需求的同时，也留有后续迭代的余地。wangsai
* 一个知识点比较难，别人讲过几次还是没理解，那么再次请教的时候，先说出自己会的东西，然后用新的角度提问自己不会的东西。yutian
* 考虑兼容性，reroute_version_update 的增加和 rsync 的删除都要考虑到系统可能还有旧版本在运行。yutian
* 所有的策略讨论的时候，可以尽量展开相对多的可能性，但是落地的时候一定要收敛。适当的额控制粗糙程度，即便是复杂和精细的策略落地的时候也要用简单的方式呈现在文档和代码上。重点是让人理解。yutian
* 写策略类代码宁愿笨一点，把 if else 多种情况罗列全，也不要图方便整合过多的逻辑。代码简单是面向人类逻辑的简单，不是代码数量少。yiwu
* 即使是策略类代码，也有可以创新的地方，yutian 接手的时候是没有局部化策略的，那我目前只是在已有策略上演进。yiwu
* 不做 delay project 最后一个人，先把主体功能交上去后再做 bugfix，做一个功能，要让代码改动尽可能小。yutian





Many git tutorials focus on a set of commands and instructions to “get you up to speed” in git, without addressing the underlying concept of “how git works”. While the commands are important, I feel it’s more important for you to understand what’s going on behind the scenes. (Full reference material is included in the appendix at the end of this document. Most diagrams are taken from Pro Git.)

At a high level, git can be thought of as a database/filesystem/backing store that remembers a truckload of details about your code base. This information is called the git repository, and contains three types of content: blobs, trees and commits.

necessary --> essential

important --> significant

at the same time --> simultaneously

the alternate approach，一种可选的方式

think of A in terms of B，从 B 的角度来考虑 A