categories: Note

tags:

- Web

date: 2018-09-24

toc: true

title: 代码评审的不可能三角
---

Code Review 是保证代码质量的重要手段之一，但许多研发团队中它常常由于各种原因并未得到真正的落地。为什么会这样呢？本文希望用一个非常简单的观点来理解这个现象，并据此给出一点优化的想法。

<!--more-->

## 观点
我们的观点可以用一句话概括，那就是**代码评审非常难同时满足高覆盖率、强约束力和低开销这三个条件**。这三个条件分别有什么含义呢？

* **高覆盖率**，意味着评审需要覆盖项目中几乎所有的提交，而不是只评审新人的代码或者是批改暑假作业般的随机「抽查」。
* **强约束力**，意味着在保证评审本身质量的基础上，评审中指出的问题都需要得到切实的解决，否则不应合并代码或发布正式版本。
* **低开销 (overhead)**，意味着评审不应占用过多宝贵的开发时间，更不应像某些会议那样提起来就让人皱眉头。


## 论证
满足上述三个条件的代码评审，应该是每一位对代码质量有追求的开发同学都不会排斥的。但为什么我们认为这样的评审可行性不高呢？简单地组合一下上述的条件，就不难发现矛盾了。

* 同时满足**高覆盖率**和**强约束力**的评审，时间是不可控的：为一些强调代码质量的开源项目贡献过 non-trivial 代码的同学，应该都知道即便是一个简单的 fix，其 PR 都可能因为实现手法和维护者的理解有偏差而长期保持在 Open 状态（俗称合不进去），更别说全新的特性与 API 了。
* 同时满足**高覆盖率**和**低开销**的评审，很容易流于形式：如果制度上约定必须对全部代码做评审，又不能耽误版本进度，那么这时候只要时间稍微一紧，评审就会变成日常回复 LGTM (Look Good To Me) 的走过场了。
* 同时满足**强约束力**和**低开销**的评审，很难覆盖到全部的代码库：一个版本中通常会有一些全新的特性。如果评审者并未参与这个新特性的开发，那么全量评审一个新特性的上千行代码，其难度跟打开一个没有读过的开源项目并马上指出其中的 bug 差不多。


## 折中
如果上述三者不可得兼，我们应该如何权衡呢？在目前的大环境下，多数软件项目快速迭代的性质与我们对「早点下班」的渴望，使得**低开销**这一条件通常很难被牺牲。那么在**覆盖率**和**约束力**之间该如何取舍呢？让我们回到「代码评审有什么作用」这个话题上吧。我们知道代码评审可以：

* 减少代码中的暗坑
* 提高被评审者的代码质量
* 让团队成员熟悉代码

如果评审本身就形同虚设，上面这些好处自然也只是空谈而已。因此，我们仍然很难放弃对**约束力**的要求。那么，如何改善这时**覆盖率**的问题呢？这里给出两点不成熟的想法供参考。

首先，来自 Google Subversion 团队的经验可以给我们一些启发：他们将代码评审与即时通信、会议、文档一起，视作团队中的**沟通方式**，而不是流程。这样，沟通方式之间就可以取长补短地提高团队效率。实际上，**在评审全新特性时「读不过来」的问题，就可以通过设计阶段的文档来缓解**：文档与评审同样是一对多的沟通，并且对文档中方案的讨论显然比直接讨论细节要更容易。一个需要 2 周时间左右开发出的全新特性，按照 `问题定义 → 基本思路 → 实现概述 → 改进优化` 的结构化方式编写的文档，其长度应该仅在千字左右，编写文档所需时间与开发时间应当不在一个量级，还能够节约在缺失文档时向其他同学当面沟通该特性的时间（当然，对那种顺手就能搞定的需求也要求文档化，就有些繁冗了）。

另外，**代码评审的覆盖率问题，还可以通过一定的提交约定来优化**。在笔者翻译的 [conventional-commits](https://www.conventionalcommits.org/zh/v1.0.0-beta.2/) 规范中，每一次提交都可以通过形如 `fix` / `feat` / `chore` / `refactor` 的不同类型来做区分，来达成细粒度的可读提交历史。那么，在评审的 Pull Request / Merge Request 粒度上，为什么不能同样地应用该规范呢？如果我们按照这种方式区分了 PR 类型，这里就有不少的想象空间：

* 可以首先将评审的资源集中在 `refactor` 与 `perf` 一类 PR 的评审上。
* 对于 `feat` 类型存在大量新代码的 PR，只需其提供了确保团队成员理解的文档，那么就只需要保证方案设计可接受，做保证代码风格一致级别的评审即可。
* 我们可以选择性忽略测试阶段可能数量众多的常规 `fix` 类 PR，但对版本发布后补充的 `hotfix` 类 PR，仍然需要评审。
* 对不影响代码质量的 `chore` 与 `doc` 类 PR 可以忽略评审。

总之，代码评审是一种沟通方式，希望它能够成为团队日常开发「文化」的一部分，而非束缚效率的死板流程。希望本文的想法对同样被评审困扰的同学有帮助 :)

> P.S. 我们 base 厦门的前端工具（编辑器）团队缺人中，有意戳[这里](https://www.zhihu.com/question/293047616/answer/497191927)了解详情哈。
