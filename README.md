Kubemark是K8s官方提供的一个对K8s集群进行性能测试的工具。它可以模拟出一个K8s cluster（Kubemark cluster），不受资源限制，从而能够测试的集群规模比真实集群大的多。这个cluster中master是真实的机器，所有的nodes是Hollow nodes。Hollow nodes执行的还是真实的K8s程序，只是不会调用Docker，因此测试会走一套K8s API调用的完整流程，但是不会真正创建pod。

Kubermark是在模拟的Kubemark cluster上跑E2E测试，从而获得集群的性能指标。Kubermark cluster的测试数据，虽然与真实集群的稍微有点误差，不过可以代表真实集群的数据，具体数据见Updates to Performance and Scalability in Kubernetes 1.3 – 2,000 node 60,000 pod clusters。因此，可以借用Kubermark，直接在真实集群上跑E2E测试，从而对我们真实集群进行性能测试。

测试设计
测试目标
集群容量上限：Node总数和Pod总数两个维度
性能瓶颈
服务延迟
服务质量目标（SLO）
Node启动时间：4s
99%的Pod启动时间：5s
99%的API调用延时：1s
服务质量指标（SLI）
指标
Pod启动时间
API调用延时
定义
集群容量：集群中有N个Node，每个Node上30个Pod，集群容量为30 * N个Pod。
集群负载：集群中Pod总数量占集群容量百分比。通过调整Pod总数控制集群负载。
百分位指标：指标的分布性。以API调用延时指标为例，50% API调用延时为20ms，90% API调用延时为50ms。
测试的APIs
Actions	Resources
PUT	Nodes，Pods，Replication Controllers
POST	Nodes，Pods，Replication Controllers
LIST	Nodes，Pods，Replication Controllers
GET	Nodes，Pods，Replication Controllers
数据统计
统计不同集群负载下，Pod启动时间和API调用延时这两个指标的分布性:

集群负载：10%， 25%，50%，90%，100%
指标分布性：10%， 50%，90%，99%
API调用延时统计数据表格示例：

 	10%-full	25%-full	50%-full	90%-full	100%-full
10th percentile	 	 	 	 	 
50th percentile	 	 	 	 	 
90th percentile	 	 	 	 	 
99th percentile	 	 	 	 	 
Optional Benchmarks
Kubemark
K8s官方提供的Benchmark，K8s自身的CI中也是采用其进行测试：ci-kubernetes-kubemark-500-gce，google-kubemark。
Cloud Container Cluster Common Benchmark
Evaluating Container Platforms at Scale中使用的Benchmark。
PerfKit Benchmarker
Google云平台提供的Benchmark，支持K8s。目前没有找到其在K8s测试的相关资料。
Prepare Environment
Kubernetes Cluster
根据自己的平台，参考官方安装文档，搭建Kubernetes Cluster。

安装依赖
yum install -y gcc.x86_64
Go开发环境
下载go1.7.4.linux-amd64.tar.gz（保持与K8s构建用的Go版本一致）
解压到/usr/local/go目录下
设置GOROOT和GOPATH环境变量 /etc/profile中添加：
 export GOPATH=/root/gocode
 export PATH=$PATH:/opt/maven/latest/bin:$GOPATH/bin:/usr/local/go/bin
使变量生效：source /etc/profile
Test Images
为了避免下载测试镜像，给Kubemark性能测试的结果带来误差，在运行Kubemark前，需要在所有K8s nodes上提前准备好这些镜像。Kubemark测试时使用的image都是GCR上面的，对于不能翻墙的Kubernetes Cluster，可以从时速云Docker Registry下载相应的镜像到本地，然后重新标签为GCR的测试image。对于甚至不能连接外网的Kubernetes Cluster，可以通过一台可以连外网的机器从时速云Docker Registry下载所有镜像，然后上传到公司内部的Docker Registry，所有K8s nodes从公司Docker Registry上下载测试image，然后重新标签为GCR的测试image。

Run Kubemark
The version of Kubernetes source code is V1.5.3.
Git clone K8s源码
 git clone https://github.com/kubernetes/kubernetes.git
 cd kubernetes
 git checkout -b v1.5.0 v1.5.0 （checkout需要测试的K8s版本）
 cd ..
 mkdir $GOPATH/src/k8s.io
 mv kubernetes $GOPATH/src/k8s.io/
 cd $GOPATH/src/k8s.io/kubernetes
备注：后续的所有命令操作，都是在$GOPATH/src/k8s.io/kubernetes目录下。
准备构建依赖
 # cat vendor/k8s.io/client-go
 ../../staging/src/k8s.io/client-go
 # cd vendor/k8s.io/
 # ln -s -f ../../staging/src/k8s.io/client-go client-go
自动产生需要的代码bindata.go
 # go get -u github.com/jteeuwen/go-bindata/...
 # ./hack/generate-bindata.sh
构建测试文件
 make WHAT='test/e2e/e2e.test'
 make ginkgo
设置环境变量
 export KUBECTL_PATH=/bin/kubectl
 export KUBERNETES_PROVIDER=local
创建kubeconfig
 kubectl config view -o yaml >> /root/.kube/config
执行performance test
 go run hack/e2e.go -v -test  --test_args="--host=http://127.0.0.1:8080 --ginkgo.focus=\[Feature:Performance\]" >> logs/log.txt （将log重定向到文件中，方便分析数据）
非K8s master上运行Kubemark，除了–test_args中的–host需要配置为正确的apiserver，还要在跑Kubemark的机器上做如下配置（Kubemark需要通过kubectl读取kubeconfig来获得测试K8s cluster信息）：

需要有kubectl binary，存放的位置与$KUBECTL_PATH保持一致
配置kubeconfig file
  # cat ~/.kube/config
  apiVersion: v1
  clusters:
  - cluster:
      api-version: v1
      server: http://k8s-apiserver.com:8080 （测试apiserver地址）
    name: e2e-cluster
  contexts:
  - context:
      cluster: e2e-cluster
    name: e2e-context
  current-context: "e2e-context"
  kind: Config
  preferences: {}
  users: []
Trouble Shootings
不要将Windows上clone的K8s源码传到Linux上进行测试，否则会遇到以下坑：
1.1 *.sh has no permissions
需要对脚本文件增加可执行权限：find hack -name *.sh | xargs chmod +x
1.2 Something went wrong: encountered 2 errors: [error running kubectl version: fork/exec ./cluster/kubectl.sh: no such file or directory error running get status: fork/exec ./hack/e2e-internal/e2e-status.sh: no such file or directory]
需要对脚本文件进行dos2unix转换：find hack -name *.sh | xargs dos2unix
1.3 构建e2e.test测试文件失败（k8s issue）
/gofiles.mk.stamp’, needed by `pkg/apis/extensions/v1beta1/zz_generated.conversion.go’. Stop. make: *** [generated_files] Error 2
此问题没有workaround。必须在linux平台上直接clone K8s源码。
kubectl binary找不到
 2017/02/20 17:37:36 e2e.go:722: Running: kubectl version
 It looks as if you don't have a compiled kubectl binary
	 
 If you are running from a clone of the git repo, please run
 './build/run.sh make cross'. Note that this requires having
 Docker installed.
	 
 If you are running from a binary release tarball, something is wrong.
 Look at http://kubernetes.io/ for information on how to contact the
 development team for help.
 2017/02/20 17:37:36 e2e.go:724: Step 'kubectl version' finished in 11.442469ms
 2017/02/20 17:37:36 e2e.go:722: Running: get status
 Local doesn't need special preparations for e2e tests
 It looks as if you don't have a compiled kubectl binary
 If you are running from a clone of the git repo, please run
 './build/run.sh make cross'. Note that this requires having
 Docker installed.
 If you are running from a binary release tarball, something is wrong.
 Look at http://kubernetes.io/ for information on how to contact the
 development team for help.
 2017/02/20 17:37:36 e2e.go:724: Step 'get status' finished in 15.370992ms
 2017/02/20 17:37:36 e2e.go:180: Something went wrong: encountered 2 errors: [error running kubectl version: exit status 1 error running get status: exit status 1]
 exit status 1
手动指定kubectl path: export KUBECTL_PATH=/bin/kubectl
ginkgo和e2e.test测试文件找不到
 ./hack/ginkgo-e2e.sh: line 117: : command not found
 !!! Error in ./hack/ginkgo-e2e.sh:117
   '"${ginkgo}" "${ginkgo_args[@]:+${ginkgo_args[@]}}" "${e2e_test}" -- "${auth_config[@]:+${auth_config[@]}}" --ginkgo.flakeAttempts="${FLAKE_ATTEMPTS}" --host="${KUBE_MASTER_URL}" --provider="${KUBERNETES_PROVIDER}" --gce-project="${PROJECT:-}" --gce-zone="${ZONE:-}" --gke-cluster="${CLUSTER_NAME:-}" --kube-master="${KUBE_MASTER:-}" --cluster-tag="${CLUSTER_ID:-}" --repo-root="${KUBE_ROOT}" --node-instance-group="${NODE_INSTANCE_GROUP:-}" --prefix="${KUBE_GCE_INSTANCE_PREFIX:-e2e}" --network="${KUBE_GCE_NETWORK:-${KUBE_GKE_NETWORK:-e2e}}" ${KUBE_CONTAINER_RUNTIME:+"--container-runtime=${KUBE_CONTAINER_RUNTIME}"} ${MASTER_OS_DISTRIBUTION:+"--master-os-distro=${MASTER_OS_DISTRIBUTION}"} ${NODE_OS_DISTRIBUTION:+"--node-os-distro=${NODE_OS_DISTRIBUTION}"} ${NUM_NODES:+"--num-nodes=${NUM_NODES}"} ${E2E_CLEAN_START:+"--clean-start=true"} ${E2E_MIN_STARTUP_PODS:+"--minStartupPods=${E2E_MIN_STARTUP_PODS}"} ${E2E_REPORT_DIR:+"--report-dir=${E2E_REPORT_DIR}"} ${E2E_REPORT_PREFIX:+"--report-prefix=${E2E_REPORT_PREFIX}"} "${@:-}"' exited with status 127
 Call stack:
   1: ./hack/ginkgo-e2e.sh:117 main(...)
 Exiting with status 1
 2017/02/20 17:45:50 e2e.go:724: Step 'Ginkgo tests' finished in 49.760083ms
 2017/02/20 17:45:50 e2e.go:180: Something went wrong: encountered 1 errors: [error running Ginkgo tests: exit status 1]
构建e2e.test：make WHAT=’test/e2e/e2e.test’
构建ginkgo：make ginkgo
构建e2e.test时，cannot find package “k8s.io/client-go/kubernetes”
 # make WHAT='test/e2e/e2e.test'
 test/e2e/disruption.go:25:2: cannot find package "k8s.io/client-go/kubernetes" in any of:
     /root/gocode/src/k8s.io/kubernetes/vendor/k8s.io/client-go/kubernetes (vendor tree)
     /usr/local/go/src/k8s.io/client-go/kubernetes (from $GOROOT)
     /root/gocode/src/k8s.io/client-go/kubernetes (from $GOPATH)
建立vendor/k8s.io/client-go的软连接：
 # cat vendor/k8s.io/client-go
 ../../staging/src/k8s.io/client-go
 # cd vendor/k8s.io/
 # ln -s ../../staging/src/k8s.io/client-go client-go
构建e2e.test时，exec: “gcc”: executable file not found in $PATH
安装GCC：yum install -y gcc.x86_64
构建e2e.test时，缺少自动产生的代码
 # k8s.io/kubernetes/test/e2e/framework
 test/e2e/framework/gobindata_util.go:30: undefined: generated.Asset
 test/e2e/framework/gobindata_util.go:33: undefined: generated.AssetNames
自动产生需要的代码：
 # go get -u github.com/jteeuwen/go-bindata/...
 # ../../../hack/generate-bindata.sh
 Generated bindata file : ../../../hack/../test/e2e/generated/bindata.go has 12697 ../../../hack/../test/e2e/generated/bindata.go lines of lovely automated artifacts
缺少kubeconfig文件
 Feb 21 18:30:25.214: INFO: >>> kubeConfig: /root/.kube/config
 F0221 18:30:25.214693   59826 e2e.go:99] Error loading client: error creating client: error loading KubeConfig: open /root/.kube/config: no such file or directory
产生kubeconfig文件：kubectl config view -o yaml » config
测试image pull不下来
先从时速云Docker Registry下载到本地，然后重新打tag，并push到公司的Docker Registry上。所有K8s nodes从公司的Docker Registry上pull需要的image，然后重新tag为测试需要个gcr image。 可以不用注册和login时速云，直接pull镜像。从时速云下载镜像过程中，偶尔出现域名解析的问题：
 # docker pull index.tenxcloud.com/google_containers/mounttest:0.7 Trying to pull repository index.tenxcloud.com/google_containers/mounttest ... Get https://index.tenxcloud.com/v2/google_containers/mounttest/manifests/0.7: Get https://rv2-ext.tenxcloud.com:5001/auth?scope=repository%3Agoogle_containers%2Fmounttest%3Apull&service=index.tenxcloud.com: dial tcp: lookup rv2-ext.tenxcloud.com: no such host
/etc/hosts中加入域名解析：119.254.210.246 rv2-ext.tenxcloud.com
kubemark运行过程中卡住，最后报”timed out waiting for the condition”
Kubemark log:
 [k8s.io] Density 
 [Feature:Performance] should allow starting 30 pods per node
 /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/density.go:641
 [BeforeEach] [k8s.io] Density
 /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/framework/framework.go:141
 STEP: Creating a kubernetes client
 Mar 3 03:12:59.815: INFO: >>> kubeConfig: /root/.kube/config
 STEP: Building a namespace api object
 [AfterEach] [k8s.io] Density
 /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/density.go:302
 Mar 3 03:14:59.822: INFO: Error building encoder: json: unsupported value: NaN
 Mar 3 03:14:59.822: INFO: Cluster saturation time: 
 [AfterEach] [k8s.io] Density
 /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/framework/framework.go:142
 STEP: Destroying namespace "e2e-tests-density-rhgjk" for this suite.
 Mar 3 03:15:09.843: INFO: namespace: e2e-tests-density-rhgjk, resource: bindings, ignored listing per whitelist
 • Failure in Spec Setup (BeforeEach) [130.038 seconds]
 [k8s.io] Density
 /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/framework/framework.go:826
 [Feature:Performance] should allow starting 30 pods per node [BeforeEach]
 /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/density.go:641
 Expected error:
 <*errors.errorString | 0xc420361ac0>: {
 s: "timed out waiting for the condition",
 }
 timed out waiting for the condition
 not to have occurred
 /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/framework/framework.go:229
Kube-apiserver log:
 Mar 3 13:57:14 mesos-master-001 kube-apiserver: I0303 13:57:14.409756 46046 panics.go:76] GET /apis/authorization.k8s.io/v1beta1/namespaces/e2e-tests-density-100-1-8xdv2/localsubjectaccessreviews: (139.12µs) 405
 Mar 3 13:57:14 mesos-master-001 kube-apiserver: goroutine 92342 [running]:
 Mar 3 13:57:14 mesos-master-001 kube-apiserver: k8s.io/kubernetes/pkg/httplog.(*respLogger).recordStatus(0xc4238e12d0, 0x195)
 Mar 3 13:57:14 mesos-master-001 kube-apiserver: /go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/pkg/httplog/log.go:219 +0xbb
 .......
 Mar 3 13:57:14 mesos-master-001 kube-apiserver: created by k8s.io/kubernetes/pkg/genericapiserver/filters.(*timeoutHandler).ServeHTTP
 Mar 3 13:57:14 mesos-master-001 kube-apiserver: /go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/pkg/genericapiserver/filters/timeout.go:80 +0x1db
 Mar 3 13:57:14 mesos-master-001 kube-apiserver: logging error output: "{\"kind\":\"Status\",\"apiVersion\":\"v1\",\"metadata\":{},\"status\":\"Failure\",\"message\":\"the server does not allow this method on the requested resource\",\"reason\":\"MethodNotAllowed\",\"details\":{},\"code\":405}\n"
原因分析：
apiserver和controller manager没有设置cert file，导致它们间无法正常通信。（手动kubectl操作都正常，只是kubemark跑时可以创建namespace，但是返回会非常慢导致超时，可能跟kubemark的实现机制有关）
解决办法：
Generate cert file：cluster/saltbase/salt/generate-cert/make-ca-cert.sh ${apiserver_ip} (注意：生产cert的脚本执行慢的原因是，需要下载easy-rsa.tar.gz，可以手动下载后放入~/kube/easy-rsa.tar.gz)
Add apiserver options：–client-ca-file=/srv/kubernetes/ca.crt –tls-cert-file=/srv/kubernetes/server.cert –tls-private-key-file=/srv/kubernetes/server.key
Add controller manager options：–root-ca-file=/srv/kubernetes/ca.crt –service-account-private-key-file=/srv/kubernetes/server.key
运行Kubemark机器与K8s master时间不同步，导致测试失败
Kubemark log:
STEP: Destroying namespace "e2e-tests-density-100-1-f3ldt" for this suite.
Mar 15 21:33:16.442: INFO: namespace: e2e-tests-density-100-1-f3ldt, resource: bindings, ignored listing per whitelist
• Failure [95.446 seconds]
[k8s.io] Density
/root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/framework/framework.go:826
  [Feature:Performance] should allow starting 30 pods per node [It]
  /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/density.go:641
  Expected error:
      <*errors.errorString | 0xc420aa2ff0>: {
          s: "too high pod startup latency 50th percentile: 9h58m15.451869496s",
      }
      too high pod startup latency 50th percentile: 9h58m15.451869496s
  not to have occurred
  /root/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/test/e2e/density.go:627
------------------------------
SSSSSSSSSSSSSSSSSSSSSSSSSSS....SSSSSSSSSS
Summarizing 1 Failure:
[Fail] [k8s.io] Density [It] [Feature:Performance] should allow starting 30 pods per node
安装ntp并同步两台机器的时间。
Reference
Kubernetes测试
单机运行k8s以及e2e
Evaluating Container Platforms at Scale
Measuring Node Performance
Scheduler Performance Test
Kubernetes Performance Measurements and Roadmap
Updates to Performance and Scalability in Kubernetes 1.3 – 2,000 node 60,000 pod clusters
1000 nodes and beyond: updates to Kubernetes performance and scalability in 1.2
Improving Kubernetes Scheduler Performance
Scale and Performance Testing of Kubernetes
