---
title: Building My Second Brain
layout: post
excerpt: 记录些知识管理的想法。
categories: Essay
---

{% include toc %}

随着工作年限的增长，接触、学习过的知识和解决过的问题也越来越多，但只靠人脑是很难记住那么多信息的，比如现在问我 redis 相关的东西，我大概率回答不上来。我的博客在这方面起到了很大的作用，但博客的内容是线性且完整的，不适合修改和记录碎片化的知识，比如某个小问题的解决办法、code snippets 等，这些我会用单独的 markdown 文件来记录，通常是一个领域一个文件。

除了记录知识外，markdown 文件还被用来记录 working doc。我的 working doc 以周为单位，会记录 todo list 和所有有用的信息。工作上不可避免地需要多线程工作，记这些能显著降低上下文切换的开销，还可以让我清空大脑，不用担心忘记某些事情。对于每项值得记录的 todo，我会把信息、思路、过程、结果都记录下来，这样既可以帮我理清思路，也记录了很多有价值的信息。对我来说，working doc 是个很好的知识库，理应也是知识管理的一环，这也是我不用 todoist 之类工具的原因。

我越来越意识到知识管理的重要性，它能帮助我们记录时间，让自己随着时间和经历的增长而有所积累。每项投入过不少时间和精力的事情都值得记录下来，总会有些学到的东西，而且我们肯定会在不同时间点解决过相同的问题，为了避免每次解决问题都付出相同的代价、让自己解决问题的能力和效率能随着时间增长，知识管理也是必不可少的，这大概也是 PingCAP 搞 knowledge base 的原因。但我在知识管理方面还有很多问题需要解决，比如：

- 随着 markdown 文件数量的增多、单个 markdown 文件的内容越来越多也越来越混乱，想要发挥它们的价值也越来越难。
- 书看得越来越多，纸质书太难管理和检索就迁移到了电子书，但仍避免不了看得越多忘得越多的问题。
- 突发的灵感和想法经常转瞬即逝、碎片时间看到的有启发或有用的信息等也没有好的方法来记录和整理。
- 过去的实战经验非常有价值，且很难从其他地方获得，是属于个人的独特的经验和优势，但这些都只记在了脑子里。

就如[《程序员修炼之道》](https://book.douban.com/subject/35006892/)的作者 Andrew Hunt 说的：“你应该像管理金融投资组合一样来管理知识”，我开始学习其他人的经验、构建自己的知识管理体系。

## 笔记的方法

通俗点说，知识管理就是记笔记，关于这方面的书有很多，简单介绍下我看过的。

如果让我只推荐一本书，我会推荐[《打造第二大脑》](https://book.douban.com/subject/36636224/)，因为它简单、具备可行性，也能唤醒你知识管理的意识。这本书提出的笔记方法是 **CODE** —— 抓取(Capture)、组织(Organize)、提炼(Distill)、表达(Express)。第一步是记录那些有启发的、实用的、新奇的、个性化的、让你有共鸣的信息。第二步是以行动为导向来组织信息，书中建议的信息组织方法是 **PARA**：P(Project) 记录短期的、当前你正在做的项目，笔记系统能够立刻发挥作用；A(Area) 记录长期的、会不断投入精力的领域；R(Resource) 记录感兴趣的、具有潜在参考价值的资源；A(Archive) 是前面三类中不再活跃内容的归档。第三步是加工提炼第一步快速存储下来的信息，降低笔记的体量、提升可见性。最后一步是利用记录下来的“半熟素材”进行表达和创造。书中还介绍了些记笔记的注意事项和习惯，比如记录上下文、定期回顾和总结等，就不再一一介绍了。

[《卡片笔记写作法》](https://book.douban.com/subject/35503571/) 是另一本可以一看的书，它的书名翻译的很奇怪，原作名是 *How to Take Smart Notes*，介绍的是卢曼的卡片盒笔记法，不清楚的还以为是讲写作方面的书。它和《打造第二大脑》是不同的流派，如果说第二大脑是以组织和分类为核心的、top-down 的方法，那么卡片盒笔记法就是以笔记为核心的、bottom-up 的方法。卡片盒笔记法的核心思想在于单个卡片（笔记）是最基本的管理单元，随着卡片数量的增加，卡片间的联系、知识体系的脉络会渐渐生长出来且能够不断变化。而以分类为核心的方法，笔记与笔记之间的联系在一开始就确定了，很难发现跨领域的、令人意想不到的联系，且随着笔记数量的增加，笔记的分类和管理会越来越困难。卡片盒笔记法将笔记分为三种：fleeting notes、literature notes 和 permanent(atomic) notes。所有笔记在一开始都属于 fleeting notes，不需要考虑记录的形式和格式，只需要记下来，会被进一步处理成为后两种笔记或者被删掉。literature notes 是用自己的语言记录下来的内容来自其他资源的笔记。permanent notes 是用精确、清晰和简短的语言记录下来的完整的笔记。后两种笔记进入卡片盒时需要精心挑选关键词、思考和建立与其他笔记的联系。

[《笔记的方法》](https://book.douban.com/subject/36615020/)是 [flomo](https://flomoapp.com/) 创始人写的书，内容相当于是在用 flomo 践行《打造第二大脑》的方法。

前面提到过 working doc 也是我知识管理的一部分，但它和笔记又有所不同。有趣的是前面几本书或多或少也提到过类似的观点，于是我就又看了两本这方面的书[《子弹笔记》](https://book.douban.com/subject/30395230/)和[《搞定》](https://book.douban.com/subject/26612471/)。这两本书其实就是在说记 working doc 或者说 journal 的价值，其中前者完全不推荐看，后者如果你还是初入职场的阶段，那可以看看，里面还有些做事方法的建议，但也不是很推荐。现在这两本书的内容我只记得 GTD 里的两句话：两分钟法则，两分钟能完成的事情立刻去做；移出工作篮的事情不能再放回工作篮。

## 笔记工具的选择

市面上的笔记工具层出不穷，笔记工具本身也体现了它认可的笔记的方法。我接触过的笔记工具有 [notion](https://www.notion.so/)、[logseq](https://logseq.com/)、[obsidian](https://obsidian.md/) 和 [flomo](https://flomoapp.com/)，当然，还有最原始的目录加 markdown 文件。这些工具各有优劣，我最终选择了 logseq 搭配 flomo：logseq 是主要的笔记工具，只在 PC 端使用；flomo 只用来在移动端记录 fleeting notes，然后在 PC 端整理进 logseq。

对于笔记工具，我有两点不能妥协的要求：一是支持本地存储，二是使用通用格式。本地存储是为了数据安全性，避免数据丢失，虽然需要自己搞备份。通用格式是为了笔记的可迁移性，避免被单个工具绑架。这两点就把 notion 排除在外了。目录加 markdown 文件的方式太过原始，缺少很多现代笔记工具的功能，比如双链、tag 等，更别提丰富的插件了，所以也被我淘汰了。logseq 和 obsidian 都是很好的笔记工具，它们的设计理念不太一样，可以按需选择甚至搭配使用：

- obsidian 更符合传统的笔记方式，它在目录加 markdown 文件的组织方式基础上加入了双链、tag 等功能，再搭配上非常丰富的插件生态，可以实现很多高级的功能。
- logseq 在设计上就很不同，它的基本管理单元是 block 而不是 page。block 可以简单理解为 logseq 中每段话都是 markdown list 的一个 item，item 和它的子 item 构成了一个 block，page 就是 block 的集合。logseq 还没有文件夹的概念，打开会默认进入 journal 页面，且默认每天会自动创建当天的 journal。

obsidian 非常适合 PARA 体系，而 logseq 更像是卡片盒笔记法。我选择 logseq 的主要原因是它的 journal 使用体验更好，我笔记的主要来源就是 journal 或者说 working doc，logseq 在单 page 上的 task 管理和 outline 方面的使用体验都更好，即使 obsidian 能通过插件实现类似的功能，仍与 logseq 有质的差距。这些差距与 logseq 的设计理念密不可分，从默认使用 journal 上也可见一斑。但 logseq 也不是没有缺点的，比如 page 管理困难、用起来卡、不适合写长文、文件的可迁移性比 obsidian 差、插件没有 obsidian 丰富等，这些都是 obsidian 的优势。所以也有人两者搭配起来使用，logseq 用来记 journal，obsidian 用来记笔记。我目前尽量 all in one，所以只用了 logseq。

flomo 我把它当做更高级的备忘录，可以给笔记打 tag 来分类，只用来记录 fleeting notes 和生活方面没那么重要的笔记。

## 我的工作流

终于到我的工作流了，不会介绍的很细节。我的工作流非常简单：

1. 每周一新建当周的 journal 用来记录当周所有的笔记。不用 daily journal 的原因是 daily 粒度太细，一天做不了什么事情。
2. 网上看到的信息会用 [omnivore](https://omnivore.app) 同步到 journal 中。看书目前用 ipad + 微信读书，不用 logseq 的 [PDF annotation](https://docs.logseq.com/#/page/pdf%20highlights) 功能，因为用电脑看书太累。微信读书里的笔记都是 fleeting notes，看完后再整理到 logseq 中。
3. journal 中的笔记都是 fleeting notes，有明确主题的笔记会用双链链接到对应的 page。周末再将当周 journal 和 flomo 中的 fleeting notes 整理成 literature notes 和 permanent notes 放到对应的 page 中或者链接起来。
4. 使用 PAR 体系管理 page，去掉了 Archive，因为没必要。logseq 虽然没有文件夹，但可以通过 template、tag、query 和 [Favorite Tree](https://github.com/sethyuan/logseq-plugin-favorite-tree) 插件实现类似的功能。使用 logseq 非常容易出现 page 泛滥的问题，而 tag 和双链在 logseq 中是等价的，都会创建 page，所以我会尽量克制 tag 的使用，且每个 tag 我都会明确它的意义和使用场景。

熟悉 LSM-tree 的人可能会发现，这就是 log + compaction 的思想。journal 就是 log，让记笔记变得简单、无痛，然后再花费些代价重写笔记，让笔记变得完整、有序。

## 写在最后

我上学时是从不记笔记的，也见到过一些笔记记得很好但效果不太好的人，现在才知道这和[必要难度理论](https://en.wikipedia.org/wiki/Desirable_difficulty)有关，记笔记的目的是为了提取，而提取的效果和记笔记的难度成正比。这个理论也能回答有了 AI 为什么还要记笔记的问题，AI 能提高信息的获取效率，但学习和提取还是要靠自己的努力。记笔记也和刻意练习有关，刻意练习是需要反馈的，而记笔记就是在记录反馈。最后，以一句很有意思的话作为结尾，希望你也能从中得到一些启发：

> 所有未保存的进度将会丢失。 —— 任天堂电子游戏“退出界面”提示
