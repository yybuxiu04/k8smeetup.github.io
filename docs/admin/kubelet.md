---
title: kubelet
notitle: true
---
## kubelet




### 概要



kubelet是运行在每个节点上的主要的“节点代理”。kubelet按照一个PodSpec工作。PodSpec是用来描述一个容器组的YAML或者JSON对象。
kubelet从各种机制（主要apiserver）获取一组PodSpec并保证在这些PodSpecs中描述的容器健康运行。kubelet不管理不是由Kubernetes创建的容器。

除了来自apiserver的PodSpec，还有三种方式可以将容器清单提供给Kubelet。

文件：路径在命令行上作为一个标识，在这个路径下的文件将被定期监视更新，监视周期默认是20秒并可以配置。

HTTP端点：HTTP端点在命令行上作为一个参数，该端点每20秒被检查一次并且检查周期可以配置。

HTTP服务：kubelet还可以监听HTTP服务并响应一个简单的API来提交一个新的清单。

```
kubelet
```

### 选项

```
      --address ip                                              Kubelet服务监听的IP地址（默认0.0.0.0，监听所有IP）
      --allow-privileged                                        设置true将允许容器请求特权模式
      --anonymous-auth                                          允许匿名请求到Kubelet服务器。未被另一个身份验证方法拒绝的请求被视为匿名请求。匿名请求包含系统的用户名:anonymous，以及系统的组名:unauthenticated。（默认 true）
      --authentication-token-webhook                            使用TokenReview API来确定不记名令牌的身份验证
      --authentication-token-webhook-cache-ttl duration         webhook令牌身份验证器缓存响应时间。（默认2m0s）
      --authorization-mode string                               Kubelet服务器的授权模式。有效的选项是AlwaysAllow或Webhook。Webhook模式使用SubjectAccessReview API来确定授权。(默认“AlwaysAllow”)
      --authorization-webhook-cache-authorized-ttl duration     来自webhook的已认证响应缓存时间（默认5m0s）
      --authorization-webhook-cache-unauthorized-ttl duration   来自webhook的未认证响应缓存时间（默认30s）
      --azure-container-registry-config string                  Azure容器注册表配置信息路径
      --bootstrap-kubeconfig string                             用于获取kubelet客户端证书的kubeconfig文件路径，如果由--kubeconfig指定的文件不存在，将使用bootstrap kubeconfig 从API服务器请求一个客户端证书，成功后，引用生成证书文件和密钥的kubeconfig将被写入--kubeconfig指定的文件，客户端证书和密钥将被保存在--cert-dir指定的目录。
      --cadvisor-port int32                                     本地cAdvisor端点的端口（默认 4194）
      --cert-dir string                                         TLS证书所在目录。如果--tls-cert-file和--tls-private-key-file指定的文件存在，当前配置将被忽略。（默认“/var/run/kubernetes”）
      --cgroup-driver string                                    kubelet用来操作主机上的cgroups驱动，可选值有：“cgroupfs”和“systemd”（默认“cgroupfs”）
      --cgroup-root string                                      可选的根cgroup用于pods， 这是由容器运行时在最好的工作基础上处理的，默认：''，也就是使用容器运行时的默认值。
      --cgroups-per-qos                                         开启创建QoS cgroup层级，如果设置为true将创建顶级QoS和容器cgroups。（默认true）
      --chaos-chance float                                      如果大于0.0，引入随机客户端错误及延迟，用来测试。
      --client-ca-file string                                   如果设置，任何带有client-ca-file中签名的客户端证书的请求都将通过与客户端证书CommonName对应的标识进行身份认证。
      --cloud-config string                                     云提供商的配置文件路径，没有配置文件时为空字符串。
      --cloud-provider string                                   云服务提供商。默认情况下，kubelet将尝试自动检测云提供商，如果不使用云提供商可以指定该参数为空字符串。（默认“auto-detect”）
      --cluster-dns stringSlice                                 DNS服务器的IP地址列表，逗号分隔。这个值是用于配置指定了“dnsPolicy=ClusterFirst”的容器DNS服务器。注意：列表中所有的DNS服务器必须提供相同的记录值，否则集群中的名称解析可能无法正常工作，也就是无法确保连接DNS服务器提供正确的名称解析。
      --cluster-domain string                                   集群域名。如果设置，kubelet将配置所有容器除了主机搜索域还将搜索当前域。
      --cni-bin-dir string                                      <警告:Alpha特性>搜索CNI插件二进制文件的完整路径。默认值：/opt/cni/bin
      --cni-conf-dir string                                     <警告:Alpha特性> 搜索CNI插件配置文件的完整路径。默认值：/etc/cni/net.d
      --container-runtime string                                运行时使用的容器。可选值：‘docker’和‘rkt’。（默认‘docker’）
      --container-runtime-endpoint string                       [实验]远程运行服务的端点。目前在Linux上支持unix套接字，在windows上支持tcp。例如：‘unix:///var/run/dockershim.sock’，‘tcp://localhost:3735’（默认‘unix:///var/run/dockershim.sock’）
      --containerized                                           在容器中运行kubelet的实验支持。用于测试。
      --contention-profiling                                    如果启用了分析，启用锁争用分析。
      --cpu-cfs-quota                                           为指定CPU限制的容器强制限制CPU CFS配额(默认为true)
      --docker-disable-shared-pid                               当运行1.13.1或更高版本的docker时，容器运行时接口(CRI)默认在同一个pod中的容器使用一个共享的PID名称空间，将此标志设置为对独立的PID名称空间用来恢复先前的行为，这个功能将在未来的Kubernetes发布版本移除。
      --docker-endpoint string                                  用来通信的docker端点。（默认“unix:///var/run/docker.sock”）
      --enable-controller-attach-detach                         允许附加/分离控制器来管理调度到当前节点的附加/分离卷，并禁用kubelet执行任何附加/分离操作(默认为true)
      --enable-custom-metrics                                   支持收集自定义指标。
      --enable-debugging-handlers                               开启服务用来收集日志及本地运行的容器及命令（默认true）
      --enable-server                                           开启Kubelet服务（默认true）
      --enforce-node-allocatable stringSlice                    由kubelet执行的节点分配执行级别列表，逗号分隔。可选项有 'pods', 'system-reserved' 和 'kube-reserved' 。如果指定后两种，必须同时指定 '--system-reserved-cgroup' 和 '--kube-reserved-cgroup'。 查看 https://git.k8s.io/community/contributors/design-proposals/node-allocatable.md 获取更多细节。 (默认 [pods])
      --event-burst int32                                       一个突发事件记录的最大值。 仅当设置--event-qps 大于0时，暂时允许该事件记录值超过设定值，但不能超过event-qps的值。（默认10）
      --event-qps int32                                         设置为大于0的值，将限制每秒创建的事件数目最大为当前值，设置为0则不限制。（默认为5）
      --eviction-hard string                                    一个清理阈值的集合（例如 memory.available<1Gi），达到该阈值将触发一次容器清理，(默认“memory.available < 100 mi,nodefs.available < 10%,nodefs.inodesFree < 10%)
      --eviction-max-pod-grace-period int32                     满足清理阈值时，终止容器组的最大响应时间，如果设置为负值，将使用pod设定的值。.
      --eviction-minimum-reclaim string                         一个资源回收最小值的集合（例如imagefs.available=2Gi），即kubelet压力较大时 ，执行pod清理回收的资源最小值。
      --eviction-pressure-transition-period duration            过渡出清理压力条件前，kubelet需要等待的时间。默认（5m0S）
      --eviction-soft string                                    一个清理阈值的集合（例如 memory.available<1.5Gi），如果达到一个清理周期将触发一次容器清理。
      --eviction-soft-grace-period string                       一个清理周期的集合（例如 memory.available=1m30s），在触发一个容器清理之前一个软清理阈值需要保持多久。
      --exit-on-lock-contention                                 kubelet是否应该退出锁文件争用。
      --experimental-allocatable-ignore-eviction                设置为true，计算节点分配时硬清理阈值将被忽略。查看 https://git.k8s.io/community/contributors/design-proposals/node-allocatable.md 获取更多细节。[默认false] 
      --experimental-allowed-unsafe-sysctls stringSlice         不安全的sysctls或者sysctl模式（以*结尾）白名单列表，以逗号分隔。在自己的风险中使用这些。
      --experimental-bootstrap-kubeconfig string                已过时：使用--bootstrap-kubeconfig
      --experimental-check-node-capabilities-before-mount       [实验]如果设置为true,kubelet将在执行mount之前检查基础节点所需组件(二进制文件等)。
      --experimental-fail-swap-on                               如果在节点上启用了swap，Kubelet将启动失败，这是一个维护遗留行为的临时选项，在v1.6启动失败是因为默认启用了swap。
      --experimental-kernel-memcg-notification                  如果启用，kubelet将集成内核memcg通知以确定是否达到内存清理阈值，而不是轮询。
      --experimental-mounter-path string                        [实验]二进制文件的挂载路径。保留空以使用默认。
      --experimental-qos-reserved mapStringString               一个资源占比的集合（例如，memory=50%），描述如何在QoS级别保留pod资源请求，目前仅支持内存。[默认none]
      --feature-gates string                                    一组键值对，用来描述alpha或实验特性，选项有：
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
      --file-check-frequency duration                           检查新数据配置文件的周期（默认20s）
      --google-json-key string                                  用于谷歌云平台服务帐户身份验证的JSON密钥。
      --hairpin-mode string                                     kubelet如何设置hairpin NAT（“发夹”转换）。 这使得当服务可以尝试访问自己时服务端点可以自动恢复，合法值由 "promiscuous-bridge", "hairpin-veth" 和 "none". (默认 "promiscuous-bridge")
      --healthz-bind-address ip                                 健康检查服务的IP地址。（设置0.0.0.0使用所有地址）（默认127.0.0.1）
      --healthz-port int32                                      本地健康检查服务的端口号（默认10248）
      --host-ipc-sources stringSlice                            Kubelet允许pod使用主机ipc名称空间列表，逗号分隔。（默认[*]）
      --host-network-sources stringSlice                        Kubelet允许pod使用主机网络列表，逗号分隔。（默认[*]）
      --host-pid-sources stringSlice                            Kubelet允许pod使用主机pid名称空间列表，逗号分隔。（默认[*]）
      --hostname-override string                                如果不是空，将使用该字符作为hostname而不是真实的hostname。
      --http-check-frequency duration                           通过http检查新数据的周期（默认20s）
      --image-gc-high-threshold int32                           镜像占用磁盘比率最大值，超过此值将执行镜像垃圾回收。（默认85）
      --image-gc-low-threshold int32                            镜像占用磁盘比率最小值，低于此值将停止镜像垃圾回收。（默认80）
      --image-pull-progress-deadline duration                   镜像拉取进度最大时间，如果在这段时间拉取镜像没有任何进展，将取消拉取。（默认1m0s）
      --image-service-endpoint string                           [实验]远程镜像服务端点。如果没有指定，默认情况下将与容器运行时端点相同。目前在Linux上支持unix套接字，在windows上支持tcp。  例如:'unix:///var/run/dockershim.sock', 'tcp://localhost:3735'
      --iptables-drop-bit int32                                 用于标记丢弃数据包的fwmark空间位，取值范围[0，31]。(默认15)
      --iptables-masquerade-bit int32                           用于标记SNAT数据包的fwmark空间位，取值范围[0，31]，请将此参数与kube-proxy中的相应参数匹配。（默认14）
      --keep-terminated-pod-volumes                             在容器停止后保持容器卷挂载在节点上，这对调试卷相关问题非常有用。
      --kube-api-burst int32                                    与kubernetes apiserver会话时的并发数。（默认10）
      --kube-api-content-type string                            发送到apiserver的请求Content type。（默认“application/vnd.kubernetes.protobuf”）
      --kube-api-qps int32                                      与kubernetes apiserver会话时的QPS。（默认15）
      --kube-reserved mapStringString                           一个资源预留量的集合（例如 cpu=200m,memory=500Mi, storage=1Gi） ，即为kubernetes系统组件预留的资源，目前支持根文件系统的cpu、内存和本地存储。查看 http://kubernetes.io/docs/user-guide/compute-resources 或许更多细节。[默认none]
      --kube-reserved-cgroup string                             用来管理kubernetes组件的顶级cgroup的绝对名称，这些组件用来管理那些标记‘--kube-reserved’的计算资源。 [默认'']
      --kubeconfig string                                       kubeconfig文件的路径，用来指定如何连接到API server，此时将使用--api-servers 除非设置了--require-kubeconfig。（默认“/var/lib/kubelet/kubeconfig”）
      --kubelet-cgroups string                                  可选的cgroups的绝对名称来创建和运行Kubelet。
      --lock-file string                                        <警告:Alpha特性> kubelet用于锁文件的路径。
      --make-iptables-util-chains                               如果为true，kubelet将确保iptables工具规则在主机上生效。（默认true）
      --manifest-url string                                     访问容器清单的URL。
      --manifest-url-header string                              访问容器清单的URL的HTTP头，key和value之间用:分隔。
      --max-open-files int                                      Kubelet进程可以打开的文件数目。（默认 1000000）
      --max-pods int32                                          当前Kubelet可以运行的容器组数目。（默认 110）
      --minimum-image-ttl-duration duration                     在执行垃圾回收前未使用镜像的最小年龄。例如： '300ms', '10s' or '2h45m'. (默认 2m0s)
      --network-plugin string                                   <警告:Alpha特性> 在kubelet/pod生命周期中为各种事件调用的网络插件的名称
      --network-plugin-mtu int32                                <警告:Alpha特性> 传递给网络插件的MTU值以覆盖缺省值，设置为0将使用默认值1460。
      --node-ip string                                          当前节点的IP地址，如果设置，kubelet将使用这个地址作为节点ip地址。
      --node-labels mapStringString                             <警告:Alpha特性> 在集群中注册节点时添加的标签，标签必须为用英文逗号分隔的key=value对。
      --node-status-update-frequency duration                   指定kubelet的节点状态为master的频率。注意:在修改时要小心，它必须与nodecontroller的nodeMonitorGracePeriod 一起工作。(默认10s)
      --oom-score-adj int32                                     kubelet进程的oom-score-adj值，范围[-1000, 1000] (默认 -999)
      --pod-cidr string                                         用于pod IP地址的CIDR，仅在单点模式下使用。在集群模式下，这是由master获得的。
      --pod-infra-container-image string                        每个pod中的network/ipc名称空间容器将使用的镜像。 (默认 "gcr.io/google_containers/pause-amd64:3.0")
      --pod-manifest-path string                                包含pod清单文件的目录或者单个pod清单文件的路径。从点开始的文件将被忽略。
      --pods-per-core int32                                     可以在这个kubelet上运行的容器组数目，在这个Kubelet上的容器组数目不能超过max-pods，所以如果在这个kubelet上运行更多的容器组应同时使用max-pods，设置为0将禁用这个限制。
      --port int32                                              Kubelet服务的端口 (默认 10250)
      --protect-kernel-defaults                                 kubelet的默认内核调优行为。设置之后，kubelet将在任何可调参数与默认值不同时抛出异常。
      --provider-id string                                      在机器数据库中标识节点的唯一标识符，也就是云提供商
      --read-only-port int32                                    没有认证/授权的只读kubelet服务端口。 (设置为0以禁用) (默认 10255)
      --really-crash-for-testing                                设置为true，when panics occur crash. 用于测试。
      --register-node                                           用apiserver注册节点 (如果设置了--api-servers默认为true) (默认 true)
      --register-with-taints []api.Taint                        用给定的列表注册节点 (逗号分隔 "<key>=<value>:<effect>")。如果register-node为false将无操作
      --registry-burst int32                                    拉去镜像的最大并发数，允许同时拉取的镜像数，不能超过registry-qps，仅当--registry-qps大于0时使用。 (默认 10)
      --registry-qps int32                                      如果大于0，将限制每秒拉去镜像个数为这个值，如果为0则不限制。 (默认 5)
      --require-kubeconfig                                      设置为true，kubelet将在配置错误时退出并忽略--api-servers指定的值以使用在kubeconfig文件中定义的服务器。
      --resolv-conf string                                      用作容器DNS解析配置的解析器配置文件。 (默认 "/etc/resolv.conf")
      --rkt-api-endpoint string                                 与rkt API 服务通信的端点，仅当设置--container-runtime='rkt'时有效 (默认 "localhost:15441")
      --rkt-path string                                         rkt二进制文件的路径，设置为空将使用$PATH中的第一个rkt，仅当设置--container-runtime='rkt'时有效。
      --root-dir string                                         管理kubelet文件的目录 (卷挂载等). (默认 "/var/lib/kubelet")
      --runonce                                                 如果为true，将在从本地清单或者远端url生成容器组后退出，除非指定了--api-servers和--enable-server
      --runtime-cgroups string                                  可选的cgroups的绝对名称，创建和运行时使用。
      --runtime-request-timeout duration                        除了pull, logs, exec 和 attach这些长运行请求之外的所有运行时请求的超时时间。 当到达超时时间，kubelet将取消请求，抛出异常并稍后重试。 (默认 2m0s)
      --seccomp-profile-root string                             seccomp配置文件目录。 (默认 "/var/lib/kubelet/seccomp")
      --serialize-image-pulls                                   一次拉取一个镜像。建议在安装docker版本低于1.9的节点或一个Aufs存储后端不去修改这个默认值。查看问题 #10959 获取更多细节。 (默认 true)
      --streaming-connection-idle-timeout duration              在连接自动关闭之前，流连接的最大空闲时间，0表示永不超时。例如： '5m' (默认 4h0m0s)
      --sync-frequency duration                                 同步运行容器和配置之间的最大时间间隔 (默认 1m0s)
      --system-cgroups /                                        可选的cgroups的绝对名称，用于将未包含在cgroup内的所有非内核进程放置在根目录 / 中，回滚这个标识需要重启。
      --system-reserved mapStringString                         一个 资源名称=量 的集合(例如 cpu=200m,memory=500Mi) 用来描述为非kubernetes组件保留的资源。 目前仅支持cpu和内存。 查看 http://kubernetes.io/docs/user-guide/compute-resources 或许更多细节。 [默认=none]
      --system-reserved-cgroup string                           顶级cgroup的绝对名称，用于管理计算资源的非kubernetes组件，这些组件通过'--system-reserved'标识保留系统资源。除了'/system-reserved'。 [默认'']
      --tls-cert-file string                                    包含用于https服务的x509证书的文件 (中间证书，如果有，在服务器认证后使用)。如果没有提供 --tls-cert-file 和 --tls-private-key-file， 将会生产一个自签名的证书及密钥给公开地址使用，并将其保存在--cert-dir指定的目录。
      --tls-private-key-file string                             包含x509私钥匹配的文件 --tls-cert-file.
      --version version[=true]                                  打印kubelet版本并退出。
      --volume-plugin-dir string                                <警告:Alpha特性> 第三方卷插件的完整搜索路径。 (默认 "/usr/libexec/kubernetes/kubelet-plugins/volume/exec/")
      --volume-stats-agg-period duration                        指定kubelet计算和缓存所有容器组及卷的磁盘使用量时间间隔。设置为0禁用卷计算。（默认1m）
```

