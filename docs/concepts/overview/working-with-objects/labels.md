---
assignees:
- mikedanese
title: Labels 和 Selectors
redirect_from:
- "/docs/user-guide/labels/"
- "/docs/user-guide/labels.html"
---

_Labels_是连结到例如像pod这样的对象的键值对.
标签是意在被用来指定那些对象(有意义的并和用户有关的)的确认属性,但对核心系统不要直接使用隐含语义.
标签能够被用来去组织和去选择对象的子集.标签能够当对象创建的时候附加上去 并且随后能够在任何时候增加和修改.
每一个对象能够有一组明确的键值对标签.每一个键必须对给定的对象是独一无二的.


```json
"labels": {
  "key1" : "value1",
  "key2" : "value2"
}
```

我们最终为了有效的查询和监视将把标签编入索引和反向索引,使用它们来给UI和CLI等排序和分组.我们不想让非确认的,尤其是大的或者有结构的数据破坏了标签.非确认的信息应该被声明使用[annotations](/docs/user-guide/annotations).



* TOC
{:toc}

## Motivation


## 动机
标签能够使用户去映射他们自己的组织结构在系统对象上以松耦合的方式并且无需客户去存储这些映射.
服务部署和批处理管道通常是多维的实体对象(例如,多个分区或者部署,多个发布轨道,多个层,每个层多个微服务).管理它们经常需要交叉操作,这打破了严格的分层表示的封装,尤其是由基础设施而不是用户决定的严格层次结构.


   * `"release" : "stable"`, `"release" : "canary"`
   * `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
   * `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
   * `"partition" : "customerA"`, `"partition" : "customerB"`
   * `"track" : "daily"`, `"track" : "weekly"`



这些只是常用标签的例子，你可以自由地开发自己的规范。请记住，给定对象的标签键必须是唯一的。


## 语法和字符集
_Labels_ 是键值对.有效的标签键有两个部分:一个可选前缀和名称,由斜杠分隔(`/`).名字部分是必需的,必须是63个字符以内,从头到尾使用一个字母数字字符(`[a-z0-9a-z]`)使用破折号(`-`),下划线(`_`),点(`.`),在字母数字之间.前缀是可选的.如果指定的话,前缀必须是DNS子域:一系列由点(`.`)分隔的DNS标签,总长度不超过253个字符,后面跟着一个斜线(`/`).
如果省略了前缀,这个标签键会被推定为用户私有.自动化的系统组件(例如. `kube-scheduler`, `kube-controller-manager`, `kube-apiserver`, `kubectl`, 或者其他第三方自动化)它增加标签到了终端用户对象上,它必须要指定一个前缀,这个`kubernetes.io/`前缀是为了Kubernetes核心组件预留的.

有效的标签值必须在63个字符以内,必须是空的或者从头到尾使用一个字母数字字符(`[ a-z0-9a-z ]`)用破折号(`-`),下划线(`_`),点(`.`),在数字字母之间.





## 标签选择器
不像 [命名和唯一标志号码](/docs/user-guide/identifiers),标签不提供唯一性.通常来说,我们希望很多对象携带相同的标签.
通过一个标签选择器,客户/用户能够识别一组对象集,标签选择器在Kubernetes中是核心分组基元.

API目前支持两种类型的选择器: 基于等式的和基于集合的.
一个标签选择器能够由多个逗号分开的需求组成.在多个需求的情况下,所有需求必须满足使用逗号分隔符作为一个_and_逻辑运算符.

空(empty)标记选择器(即零需求的选择器)选择集合中的每个对象.
空(null)标记选择器(仅适用于可选的选择器字段)不选择任何对象.

**注意**:两个控制器的标签选择器不能在命名空间内重叠,否则它们将互相竞争.




### 基于等式的需求 
等式或者基于等式的需求允许通过标签的键值来过滤.匹配对象必须满足所有指定的标签约束,尽管它们也可能有附加的标签.
这`=`,`==`,`!=`三种运算符都被承认.前两个代表等于(它们是同义词),后者代表不等于.例如:

```
environment = production
tier != frontend
```
前者选择所有的资源,其键等于`environment`和值等于`production`.
后者选择所有的资源,其键等于`tier`但值不同于`frontend`,所有的资源没有以`tier`键的标签.
这个标签能够过滤得到非`frontend`但是`production`资源使用逗号操作符:`environment=production,tier!=frontend`





### 基于集合的需求

基于集合的标签需求允许过滤键值对集合.三种操作符可以被支持:`in`,`notin` 和 exists (仅仅键标识符). 例如:

```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```
第一个例子选择所有资源,其键等于`environment`和值等于`production` 或者 `qa`.
第二个例子选择所有资源,其键等于`tier`和值不是`frontend` 和 `backend`,所有的资源不带键为`tier`的标签.
第三个例子选择所有资源,有一个键为`partition`的标签;不检查值.
第四个例子选择所有资源,没有一个键为`partition`的标签;不检查值.
同样的这个逗号分隔符充当_AND_运算.所以过滤资源的时候使用`partition`键(忽略值)和`environment`键不为`qa`通过使用`partition,environment notin (qa)`来实现.
基于集合的标签是等式的一个总的形式,`environment=production`相当于 `environment in (production)`;类似地`!=` 相当于 `notin`.

基于集合的需求能和基于等式的需求混合使用,例如:`partition in (customerA, customerB),environment!=qa`.



## API

### 列出和监视过滤
列出和监视操作使用查询参数可以指定标签选择器去过滤一个对象的集合.这两个需求是被允许的(在URL查询字符串中显示它们):

  * _equality-based_ requirements: `?labelSelector=environment%3Dproduction,tier%3Dfrontend`
  * _set-based_ requirements: `?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`


两种标签选择器样式能被用来去列出或者监视资源通过一个REST的客户端请求.例如,针对使用`kubectl`操作`apiserver`和使用基于等式的标签可以这样写:

```shell
$ kubectl get pods -l environment=production,tier=frontend
```

或者使用基于集合的请求:

```shell
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
```

正如已经提到的基于集合的要求是要更具有表现力.举例来说,它们可以在值上执行_or_操作:

```shell
$ kubectl get pods -l 'environment in (production, qa)'
```

或者通过_exists_操作符限制否定的匹配:

```shell
$ kubectl get pods -l 'environment,environment notin (frontend)'
```



### 设置API对象的参考

一些Kubernetes对象,例如 [`services`](/docs/user-guide/services)和[`replicationcontrollers`](/docs/user-guide/replication-controller),也使用标签选择器去指定其他资源的集合,例如[pods](/docs/user-guide/pods).

#### 服务和副本控制器

一个pods的集合通过一个标签选择器来被指向到到一个`service`上.同样的,一个`replicationcontroller`管理的所有pods也应该使用一个标签选择器来明确.
标签选择器对于services和replicationcontrollers这两个对象使用json或者yaml文件映射的方式来被定义.目前仅基于等式的需求选择器被支持:

```json
"selector": {
    "component" : "redis",
}
```
or

```yaml
selector:
    component: redis
```


这个选择器(分别地使用`json` 或者 `yaml` 格式)是等效于`component=redis` 或者 `component in (redis)`.

#### 资源支持基于集合的需求

kubernetes最新的资源类型,例如[`Job`](/docs/concepts/jobs/run-to-completion-finite-workloads/), [`Deployment`](/docs/concepts/workloads/controllers/deployment/), [`Replica Set`](/docs/concepts/workloads/controllers/replicaset/), and [`Daemon Set`](/docs/concepts/workloads/controllers/daemonset/),也支持基于集合的需求.

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```


`matchLabels`是`{key,value}`键值对的映射,一个单独的 `{key,value}`键值对在`matchLabels`映射里面是等价于一个`matchExpressions`的要素,它们的`key`域是"key",这个操作符是"In",这个`values`的数组仅仅包含"value".`matchExpressions`是一个pod选择器需求的列表.有效的操作符包括In, NotIn, Exists, 和 DoesNotExist.值的设置就In和NotIn来说必须是非空.所有的这些需求,从`matchLabels` and `matchExpressions`是要放一起--它们必须所有条件都要被满足后,然后目的是去匹配.

#### 选择节点的集合
一个选择使用标签的用例是去限制一个pod调度到指定节点的集合去.
请看文档[node selection](/docs/user-guide/node-selection)查看更多信息.
