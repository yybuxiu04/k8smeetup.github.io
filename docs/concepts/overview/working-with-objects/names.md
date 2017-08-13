---
assignees:
- mikedanese
- thockin
title: Names
redirect_from:
- "/docs/user-guide/identifiers/"
- "/docs/user-guide/identifiers.html"
---



在Kubernetes REST API 中所有的对象明确地通过一个Name(命名)和一个UID(唯一标识号码)来认定.
对于用户提供的非唯一性的属性, Kubernetes 提供 [labels](/docs/user-guide/labels) 和[annotations](/docs/user-guide/annotations).



## 命名

命名一般来说由客户提供. 每次仅一个给定种类的对象能有一个给定的命名(i.e., 它们在空间上是独一无二的). 但是如果你删除了一个对象, 你仍然能创建一个新的对象并使用相同的命名. 命名被用来关联在一个URL资源对象上,
例如 `/api/v1/pods/some-name`. 按照惯例 , Kubernetes 资源的命名 应该最多达到最大长度253个字符 并且由小写字母数字字符,`-`, 和 `.` 组成, 但是某些资源有更多特殊的限制条件,请查看
[标识符设计文档](https://git.k8s.io/community/contributors/design-proposals/identifiers.md) 



## 唯一标识号码

唯一标识号码是由Kubernetes自身生成. 每一个对象创建在一个kubernetes集群完整的生命周期中有一个截然不同的唯一标识号码 (i.e., 它们在时间上和空间上是独一无二的).
