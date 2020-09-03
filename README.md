### kubernetes 集群中服务高可用部署实践 
#### 测试环境
```
[root@VM-240-97-centos test]# kubectl version
Client Version: version.Info{Major:"1", Minor:"18+", GitVersion:"v1.18.4", GitCommit:"f6b0517bc6bc426715a9ff86bd6aef39c81fd64a", GitTreeState:"clean", BuildDate:"2020-08-12T02:20:58Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18+", GitVersion:"v1.18.4", GitCommit:"f6b0517bc6bc426715a9ff86bd6aef39c81fd64a", GitTreeState:"clean", BuildDate:"2020-08-12T02:18:32Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}
```

#### 1. 利用pod反亲和性配置(required)，创建一个deployment，并验证是否符合预期

文件名: nginx-pod-anti-affinity.yaml，关键字段如下：

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - nginx
    topologyKey: "kubernetes.io/hostname"
```

- 测试并验证

```bash
[root@VM-240-97-centos test]# kubectl  get node
NAME             STATUS   ROLES    AGE   VERSION
172.21.240.231   Ready    <none>   15d   v1.18.4
172.21.240.97    Ready    <none>   15d   v1.18.4
172.21.64.101    Ready    <none>   14d   v1.18.4

[root@VM-240-97-centos test]# kubectl  create -f nginx-pod-antiaffinity.yaml

[root@VM-240-97-centos test]# kubectl  get pod -o wide
NAME                                READY   STATUS              RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
nginx-deployment-6bcf4f9b96-nw6xl   1/1     Running             0          11s     172.23.0.43    172.21.240.231   <none>           <none>
nginx-deployment-6bcf4f9b96-rwmw4   1/1     Running             0          11s     172.23.0.99    172.21.240.97    <none>           <none>
nginx-deployment-6bcf4f9b96-zwxpb   1/1     Running             0          11s     172.23.0.145   172.21.64.101    <none>           <none>
```
从运行结果来看，每个pod分别调度到了不同的节点，那么如果再增加副本数呢？比如设置副本数为4:

```bash
[root@VM-240-97-centos test]# kubectl  scale deploy nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled

[root@VM-240-97-centos test]# kubectl  get pod -o wide
NAME                                READY   STATUS              RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
nginx-deployment-6bcf4f9b96-nw6xl   1/1     Running             0          13m     172.23.0.43    172.21.240.231   <none>           <none>
nginx-deployment-6bcf4f9b96-rwmw4   1/1     Running             0          13m     172.23.0.99    172.21.240.97    <none>           <none>
nginx-deployment-6bcf4f9b96-sdbt4   0/1     Pending             0          69s     <none>         <none>           <none>           <none>
nginx-deployment-6bcf4f9b96-zwxpb   1/1     Running             0          13m     172.23.0.145   172.21.64.101    <none>           <none>
```
从运行结果中会发现，新扩容出来的pod一直处于Pending状态，无法得到调度，是因为该k8s集群一共有3个node
，业务的 nginx-deployment副本数为4，并且在nginx-pod-anti-affinity.yaml中，pod antiaffinity 采用了requiredDuringSchedulingIgnoredDuringExecution这种硬性限制，必须满足调度策略才会进行调度，kube-scheduler还支持另一种软性限制 preferredDuringSchedulingIgnoredDuringExecution，尽量满足调度策略即可调度。现在替换成 preferredDuringSchedulingIgnoredDuringExecution再次测试验证：

文件名: nginx-pod-antiaffinity-preferred.yaml

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - nginx
      topologyKey: kubernetes.io/hostname
```
再次验证调度结果：

```bash
[root@VM-240-97-centos test]# kubectl  get pod -o wide
NAME                                READY   STATUS              RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
nginx-deployment-5667d75bfd-42cth   1/1     Running             0          3m25s   172.23.0.148   172.21.64.101    <none>           <none>
nginx-deployment-5667d75bfd-5nhqb   1/1     Running             0          3m25s   172.23.0.146   172.21.64.101    <none>           <none>
nginx-deployment-5667d75bfd-dp8bq   1/1     Running             0          3m25s   172.23.0.147   172.21.64.101    <none>           <none>
nginx-deployment-5667d75bfd-zrmd9   1/1     Running             0          3s      172.23.0.149   172.21.64.101    <none>           <none>
```
具体采用 requiredDuringSchedulingIgnoredDuringExecution 还是 preferredDuringSchedulingIgnoredDuringExecution调度策略限制，视当前业务需求而定即可；

#### 2. 创建一个deployment nginx，默认副本数为3，使其调度到同一个k8s节点上

使用podAffinity调度策略，可以将一个工作负载中的pod调度到同一个k8s节点上。

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
        - key: app
          operator: In
          values:
            - nginx
    topologyKey: "kubernetes.io/hostname"
```
测试结果如下：

```bash
[root@VM-240-97-centos test]# kubectl  get pod -o wide
NAME                                READY   STATUS              RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
nginx-deployment-658fb6cb8d-8gcxz   1/1     Running             0          4s      172.23.0.151   172.21.64.101    <none>           <none>
nginx-deployment-658fb6cb8d-j5txt   1/1     Running             0          4s      172.23.0.152   172.21.64.101    <none>           <none>
nginx-deployment-658fb6cb8d-ldb6x   1/1     Running             0          4s      172.23.0.150   172.21.64.101    <none>           <none>
```

#### 3.为上面的 nginx deployment 配置 PodDisruptionBudget，要求 3 个 pod 中至少有 2 个 pod 始终处于可用状态

为了保证业务能够持续稳定运行，需要将业务应用进行集群化部署。
并通过 PodDisruptionBudget 控制器设置应用POD集群处于运行状态最低个数，
也可以设置应用POD集群处于运行状态的最低百分比，这样可以保证在主动销毁应用POD的时候，
不会一次性销毁太多的应用POD，从而可以进一步提升业务的稳定性。
>MinAvailable参数：表示最小可用POD数，表示应用POD集群处于运行状态的最小POD数量，或者是运行状态的POD数同总POD数的最小百分比。

>MaxUnavailable参数：表示最大不可用PO数，表示应用POD集群处于不可用状态的最大POD数，或者是不可用状态的POD数同总POD数的最大百分比。

创建pdb:
```yaml 
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```
测试结果：

```shell script
[root@VM-240-97-centos test]# kubectl create -f minAvailiable.yaml
[root@VM-240-97-centos test]# kubectl  get pdb
NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
nginx-pdb   2               N/A               1                     5s
查看当前 nginx deployment的部署情况
[root@VM-240-97-centos test]# kubectl  get pod -o wide
NAME                                READY   STATUS              RESTARTS   AGE     IP             NODE             NOMINATED NODE   READINESS GATES
nginx-deployment-658fb6cb8d-8gcxz   1/1     Running             0          19m     172.23.0.151   172.21.64.101    <none>           <none>
nginx-deployment-658fb6cb8d-j5txt   1/1     Running             0          19m     172.23.0.152   172.21.64.101    <none>           <none>
nginx-deployment-658fb6cb8d-ldb6x   1/1     Running             0          19m     172.23.0.150   172.21.64.101    <none>           <none>
执行驱逐操作
[root@VM-240-97-centos test]# kubectl  drain 172.21.64.101
此时你会发现,pod没有发生变化,实际上是被pdb保护了
```
如果想正常删除，需要先删除PodDisruptionBudget，一般只有在部署一些分布式系统，并且需要
保证一定数量的实例可以正常运行，以免导致数据无法同步等问题，比如zk，etcd；

#### 4. 创建一个nginx  deployment，为该nginx创建nodeport service，在另外一个测试节点上，不停的发送http请求，然后进行发布更新操作，观察发压端，是否有网络超时或者服务不可用现象

为nginx-deployment 创建 nodeport svc
```shell script
[root@VM-240-97-centos test]# kubectl  get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
nginx-deployment   NodePort    172.23.254.178   <none>        80:32261/TCP   89s
```
死循环通过 nodeport 访问该nginx服务：
```shell script
[root@VM-240-97-centos test]# kubectl create -f nginx-pod-antiaffinity-preferred.yaml
[root@VM-240-97-centos test]# yum -y install httpd
发布更细之前先压测一下服务
[root@VM-240-97-centos test]# ab -n 100000 -c 10 http://172.21.64.101:32261/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 172.21.64.101 (be patient)
Completed 10000 requests
Completed 20000 requests
Completed 30000 requests
Completed 40000 requests
Completed 50000 requests
Completed 60000 requests
Completed 70000 requests
Completed 80000 requests
Completed 90000 requests
Completed 100000 requests
Finished 100000 requests


Server Software:        nginx/1.19.2
Server Hostname:        172.21.64.101
Server Port:            32261

Document Path:          /
Document Length:        612 bytes

Concurrency Level:      10
Time taken for tests:   42.226 seconds
Complete requests:      100000
Failed requests:        0
Write errors:           0
Total transferred:      84500000 bytes
HTML transferred:       61200000 bytes
Requests per second:    2368.19 [#/sec] (mean)
Time per request:       4.223 [ms] (mean)
Time per request:       0.422 [ms] (mean, across all concurrent requests)
Transfer rate:          1954.22 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    2   3.3      2    1018
Processing:     1    2   0.9      2      37
Waiting:        0    2   0.9      2      37
Total:          2    4   3.6      5    1021

Percentage of the requests served within a certain time (ms)
  50%      5
  66%      5
  75%      5
  80%      5
  90%      5
  95%      6
  98%      8
  99%      9
 100%   1021 (longest request)

执行发布更新,同时开启压测
[root@VM-240-97-centos test]# kubectl set image deploy nginx-deployment nginx=nginx:1.19
[root@VM-240-97-centos test]# ab -n 100000 -c 10 http://172.21.64.101:32261/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 172.21.64.101 (be patient)
apr_socket_recv: Connection reset by peer (104)
Total of 3980 requests completed
```
测试结果: 发现压测没有正常结束,说明本次发布更新对服务可用性是有损的，发布过程有服务不可用的情况；

#### 5.为nginx  deployment 配置 preStopHook 和 readinessProbe；重复操作5，观察是否有网络超时或者服务不可用现象；
在container配置中，加入 prestopHook 和 readinessProbe

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
readinessProbe:
tcpSocket:
  port: 80
initialDelaySeconds: 5
periodSeconds: 10
```
创建 workload, 并重复上面的步骤进行压测,再次观察压测是否被中断

```shell script
[root@VM-240-97-centos test]# kubectl create -f nginx-pod-antiaffinity-preferred.yaml
执行发布更新,同时开启压测
[root@VM-240-97-centos test]# kubectl set image deploy nginx-deployment nginx=nginx:1.19
[root@VM-240-97-centos test]# ab -n 100000 -c 10 http://172.21.64.101:32261/
[root@VM-240-97-centos test]# ab -n 100000 -c 10 http://172.21.64.101:32261/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 172.21.64.101 (be patient)
Completed 10000 requests
Completed 20000 requests
Completed 30000 requests
Completed 40000 requests
Completed 50000 requests
Completed 60000 requests
Completed 70000 requests
Completed 80000 requests
Completed 90000 requests
Completed 100000 requests
Finished 100000 requests


Server Software:        nginx/1.19.2
Server Hostname:        172.21.64.101
Server Port:            32261

Document Path:          /
Document Length:        612 bytes

Concurrency Level:      10
Time taken for tests:   36.602 seconds
Complete requests:      100000
Failed requests:        0
Write errors:           0
Total transferred:      84500000 bytes
HTML transferred:       61200000 bytes
Requests per second:    2732.06 [#/sec] (mean)
Time per request:       3.660 [ms] (mean)
Time per request:       0.366 [ms] (mean, across all concurrent requests)
Transfer rate:          2254.49 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    2   0.7      1      18
Processing:     1    2   0.8      1      19
Waiting:        0    2   0.8      1      19
Total:          2    4   1.4      3      22
WARNING: The median and mean for the initial connection time are not within a normal deviation
        These results are probably not that reliable.
WARNING: The median and mean for the processing time are not within a normal deviation
        These results are probably not that reliable.
WARNING: The median and mean for the waiting time are not within a normal deviation
        These results are probably not that reliable.

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      5
  75%      5
  80%      5
  90%      5
  95%      5
  98%      6
  99%      8
 100%     22 (longest request)
```
测试结果：通过对workload增加preStopHook 和 readinessProbe配置后，可以保证服务优雅退出和稳定启动。

#### 参考

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/  
https://kubernetes.io/docs/tasks/run-application/configure-pdb/  
https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/