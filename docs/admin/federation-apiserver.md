---title: federation-apiserver
notitle: true
---
## federation-apiserver



### 摘要


Kubernetes federation API 服务为api对象验证和配置数据，包括pods,services,
replicationcontrollers和其它组件。API服务通过各种组件的交互提供了REST操作
和集群状态的共享前端。


```
federation-apiserver
``` 


### 参数列表

```
	  --admission-control stringSlice                           用于集群资源控制管理的插件列表，以逗号隔开。(默认是 [AlwaysAdmit])
	  --admission-control-config-file string                    用于管理控制的配置文件。
	  --advertise-address ip                                    用于向集群成员开放API服务的IP地址。该地址必须可以被集群其他成员访问，如果为空，则 --bind-address 需要指定，如果--bind-address为空，则使用host的默认地址。
	  --anonymous-auth                                          允许API服务安全端口的匿名请求。匿名请求就是没有被其他验证机制拒绝的请求，它拥有系统用户名：anonymous和系统组：unauthenticated 。(默认 true)
      --audit-log-maxage int                                    最大的audit日志保存天数，以文件名包含的timestamp为准。
      --audit-log-maxbackup int                                 最多的audit日志保存数量。
      --audit-log-maxsize int                                   日志大小限制，达到这个数字将会被自动转储。
      --audit-log-path string                                   如果设置了这个参数，所有API服务收到的请求都会记录到这个文件。  '-' 表示标准输出.
      --audit-policy-file string                                定义audit策略配置的文件路径。需要AdvancedAuditing，当AdvancedAuditing为true 时，需要一个配置文件来启用auditing.
      --audit-webhook-config-file string                        定义webhook配置的kubeconfig格式的文件路径，要求'AdvancedAuditing'。




	  --audit-webhook-mode string                               发送审计事务策略的规则。Blocking表示发送的事务会禁止服务响应。Batch导致webhook异步缓存和发送事务。可选模式为：batch,blocking. (默认 "batch")
      --authentication-token-webhook-cache-ttl duration         缓存来自webhook口令验证的时间 (默认 2m0s)
      --authentication-token-webhook-config-file string         webhook用于口令验证的配置以kubeconfig格式保存的文件。API服务会向远程服务校验口令。
      --authorization-mode string                               用于在安全端口验证的plug-ins列表。以逗号隔开AlwaysAllow,AlwaysDeny,ABAC,Webhook,RBAC,Node. (默认 "AlwaysAllow")


	  
	  --authorization-policy-file string                        CSV格式的基于安全端口的验证策略文件，和--authorization-mode=ABAC一起使用。
      --authorization-webhook-cache-authorized-ttl duration     缓存来自webhook验证'authorized'的回应周期 (默认 5m0s)
      --authorization-webhook-cache-unauthorized-ttl duration   缓存来自webhook验证'unauthorized'的回应周期。(默认 30s)
      --authorization-webhook-config-file string                kubeconfig格式的webhook配置文件, 和--authorization-mode=Webhook一起使用. API服务会通过API的安全端口向远程服务校验。



	  --basic-auth-file string                                  这个文件会被用来处理通过http验证发到API服务安全端口的请求。
      --bind-address ip                                         用来监听--secure-port端口的IP地址。这个地址应该可以被集群和CLI或者网页客户端访问。如果这个地址为空，这所有的网口都会被监听，默认0.0.0.0。
      --cert-dir string                                         存放TLS证书的目录。如果配置了--tls-cert-file 和 --tls-private-key-file 则这个参数会被忽略 (默认 "/var/run/kubernetes")
      --client-ca-file string                                   任何代表由CA签名的客户证书请求将会对应客户证书的CommonName来验证。




	  --cloud-config string                                     云服务的配置文件。空代表没有使用云服务
      --cloud-provider string                                   云服务提供商的配置文件。空代表没有使用云服务
      --contention-profiling                                    启用contention profiling, 前提是profiling启用
      --cors-allowed-origins stringSlice                        逗号隔开的允许的CORS列表，可以使用正则表达式来支持子域名，如果该列表为空则CORS不启用。



      --delete-collection-workers int                           为了DeleteCollection启用的工作进程数目，用来加速清理namespace.(默认 1)
      --deserialization-cache-size int                          内存中缓存的JSON对象数量
      --enable-garbage-collector                                启用垃圾回收机制，必须跟kube-controller-manager的corresponding标志同步。(默认 true)
      --enable-swagger-ui                                       在apiserver启动swagger-ui,路径为/swagger-ui
      --etcd-cafile string                                      SSL CA列表，用于保护etcd通讯。
      --etcd-certfile string                                    SSL 证书文件路径
      --etcd-keyfile string                                     SSL 钥匙路径
      --etcd-prefix string                                      etcd里所有资源路径的前缀(默认 "/registry")
      --etcd-quorum-read                                        如果为true则启用quorum read.
      --etcd-servers stringSlice                                连接的etcd服务列表，逗号隔开 (scheme://ip:port)
      --etcd-servers-overrides stringSlice                      etcd服务资源覆盖列表, 逗号隔开。 覆盖格式为: group/resource#servers, servers格式为 http://ip:port, 逗号隔开.



      --event-ttl duration                                      保留事务的时间。 (默认 1h0m0s)
      --experimental-bootstrap-token-auth                       允许启用kube-system命名空间的'bootstrap.kubernetes.io/token'类型的secret，用于TLS bootstrapping验证。
      --experimental-encryption-provider-config string          该文件保存了加密配置用于etcd保存secret。
      --experimental-keystone-ca-file string                    该文件提供了CA列表，用于Keystone服务的证书验证。如果没有配置，则使用host的根CA。
      --experimental-keystone-url string                        启用keystone验证插件。
      --external-hostname string                                用于生成对外URL的hostname. (e.g. Swagger API Docs).
      --feature-gates mapStringBool                             描述测试或实验功能的键值对，可选项为：
Accelerators=true|false (ALPHA - default=false)
AdvancedAuditing=true|false (ALPHA - default=false)
AffinityInAnnotations=true|false (ALPHA - default=false)
AllAlpha=true|false (ALPHA - default=false)
AllowExtTrafficLocalEndpoints=true|false (default=true)
AppArmor=true|false (BETA - default=true)
DynamicKubeletConfig=true|false (ALPHA - default=false)
DynamicVolumeProvisioning=true|false (ALPHA - default=true)
ExperimentalCriticalPodAnnotation=true|false (ALPHA - default=false)
ExperimentalHostUserNamespaceDefaulting=true|false (BETA - default=false)
LocalStorageCapacityIsolation=true|false (ALPHA - default=false)
PersistentLocalVolumes=true|false (ALPHA - default=false)
RotateKubeletClientCertificate=true|false (ALPHA - default=false)
RotateKubeletServerCertificate=true|false (ALPHA - default=false)
StreamingProxyRedirects=true|false (BETA - default=true)
TaintBasedEvictions=true|false (ALPHA - default=false)



      --insecure-allow-any-token username/group1,group2         配置这个，你的服务器将变得不安全。允许任何token，而用户信息则会以username/group1,group2这样的token来解释。
      --insecure-bind-address ip                                监听--insecure-port端口的IP地址 (设置为0.0.0.0 监听所有网络接口). (默认 127.0.0.1)
      --insecure-port int                                       这个端口用于服务不安全没验证的访问。防火墙规则阻止这个端口被集群外访问，而集群的公共地址上的443端口则被转发到该端口，这在默认配置中由nginx完成。(默认 8080)
      --master-service-namespace string                         已弃用: 用于指定运行pod的kubernetes主服务器上的命名空间(默认 "default")
      --max-mutating-requests-inflight int                      指定时间内转换的请求数目，超出此数会拒绝请求，0表示没有限制。(默认 200)
      --max-requests-inflight int                               指定时间内非转换请求，超出此数会拒绝请求，0表示没有限制。(默认 400)
      --min-request-timeout int                                 非必要参数，表示一个handler保持的request的秒数。当前只用于监控request handler， 会随机挑选一个比该值大的数字作为timeout时间，来分散负载。(默认 1800)
      --oidc-ca-file string                                     如果配置了这个参数，OpenID服务的证书将由oidc-ca-file里的CA来验证，否则使用host的根CA。
      --oidc-client-id string                                   OIDC客户端ID，如果配置了oidc-issuer-url，则该值也需要配置。
      --oidc-groups-claim string                                OIDC的用户组，这个值将会是一个字符串或者字符串列。该功能正在实验当中，若需更仔细资料请查看验证文档。
      --oidc-issuer-url string                                  OpenID签发者地址，仅接受HTTPS格式，如果配置了这个参数，则会被用于验证OIDC的JSON网页口令（JWT）。
      --oidc-username-claim string                              用于OIDC的用户名，注意除了默认的（'sub'）之外不保证其唯一性，如需更多资料，请查看验证文档。
	  
	  
	
      --profiling                                               通过网页接口 host:port/debug/pprof/启用profiling (默认 true)
      --requestheader-allowed-names stringSlice                 允许--requestheader-username-headers指定的客户证书头文件，如果为空，则--requestheader-client-ca-file里CA验证的客户证书都接受。
      --requestheader-client-ca-file string                     在信任--requestheader-username-headers指定的用户名之前，使用来验证客户证书的根证书群。
      --requestheader-extra-headers-prefix stringSlice          请求头文件的前缀，建议使用X-Remote-Extra-.
      --requestheader-group-headers stringSlice                 请求头文件的组，建议使用X-Remote-Group.
      --requestheader-username-headers stringSlice              请求头文件的用户名，普遍使用X-Remote-User.
      --runtime-config mapStringString                          可能会传送给API服务的描述运行时环境的键值对。apis/<groupVersion>键可以用来开关对应的api版本。apis/<groupVersion>/<resource> 这个则用来开关对应的资源配置。api/all和api/legacy分别对应所有和主要的api版本控制。
      --secure-port int                                         提供HTTPS验证和授权的端口号，为0，则不提供HTTPS功能。(默认 6443)
      --service-account-key-file stringArray                    该文件保存了PEM编码格式x509 RSA和ECDSA的公钥或私钥，用于验证ServiceAccount口令。如果没有指定，则使用--tls-private-key-file 该文件可以包含多个钥匙，因此这个参数可以多次指定不同的文件。
      --service-account-lookup                                  如果为true,则验证过程会验证etcd里的ServiceAccount口令。 (默认 true)
      --storage-backend string                                  存储后端，可选项为: 'etcd3' (默认), 'etcd2'.
      --storage-media-type string                               存储里保存的对象格式，有些资源或者存储后端可能只支持有限的格式类型。
	  


	  --storage-versions string                              分组保存资源的版本。格式："group1/version1,group2/version2,...",当对象转移到另外一组时，只需要这样指定"group1=group2/v1beta1,group3/v1beta1,...",只需要修改变动的资源，默认是衍生自KUBE_API_VERSIONS环境变量 (默认 "admissionregistration.k8s.io/v1alpha1,apps/v1beta1,authentication.k8s.io/v1,authorization.k8s.io/v1,autoscaling/v1,batch/v1,certificates.k8s.io/v1beta1,componentconfig/v1alpha1,extensions/v1beta1,federation/v1beta1,networking.k8s.io/v1,policy/v1beta1,rbac.authorization.k8s.io/v1beta1,settings.k8s.io/v1alpha1,storage.k8s.io/v1,v1")
      --target-ram-mb int                                       api服务的内存限制 (比如用于限制缓存大小)
      --tls-ca-file string                                      该文件保存的CA会用于Admission Controllers的安全验证，文件内容必须是PEM编码的CA群，也可以在--tls-cert-file提供的证书后添加CA内容。
      --tls-cert-file string                                    该文件保存了默认的x509证书。如果HTTPS启用而--tls-cert-file and --tls-private-key-file没有配置，则会自动生成一对自签名的证书和钥匙对，并保存在/var/run/kubernetes.
      --tls-private-key-file string                             该文件保存了默认的x509私钥，对应--tls-cert-file.
      --tls-sni-cert-key namedCertKey                           x509证书和私钥路径，可选后缀是完整域名，也可能是通配符格式。如果没有提供域名，则必须提供证书名字。多个证书对，则可以多次指定--tls-sni-cert-key.比如: "example.crt,example.key" 或 "foo.crt,foo.key:*.foo.com,foo.com". (默认 [])
      --token-auth-file string                                  如果配置了这个参数，则该文件会通过口令验证来保障API服务的安全端口。
      --watch-cache                                             启用api服务的监控缓存 (默认 true)
      --watch-cache-sizes stringSlice                           每种资源的健康缓存大小，逗号隔开。格式为：resource#size, 这里size 是一个数字. 仅当watch-cache启用时该值生效。
```

###### Auto generated by spf13/cobra on 11-Jul-2017
