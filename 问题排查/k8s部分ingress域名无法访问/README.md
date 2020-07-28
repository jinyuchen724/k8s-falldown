![image](https://github.com/jinyuchen724/k8s-falldown/raw/master/%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5/%E5%AE%B9%E5%99%A8%E5%90%AF%E5%8A%A8%E7%8A%B6%E6%80%81%E5%8D%A1%E5%9C%A8ContainerCreating/1.png)

## k8s集群中部分ingress域名无法访问

如:以okr.vdian.net为例


### 查看ingress
```
#kubectl get ingress |grep okr
ep-vokr                  okr.vdian.net                                   80      34d
```

### 查看svc

```
#kubectl get svc|grep okr
ep-vokr                    ClusterIP   10.103.56.28     <none>        80/TCP     34d
```

### 查看pod

```
#kubectl get pod -o wide |grep okr
ep-vokr-0                               2/2     Running   0          34d     10.34.66.40   10.34.64.129   <none>           1/1
```

ingress和svc配置都正常,pod状态也正常

然后describe下pod的状态，发现Ready的状态是false！

```
#kubectl describe pod/ep-vokr-0
...
Conditions:
  Type                 Status
  InPlaceUpdateReady   True
  Initialized          True
  Ready                False
  ContainersReady      True
  PodScheduled         True
...

```

去对应部署服务的node节点10.34.64.129，查看下servcie ip情况发现确实被kube-proxy踢掉了。

```$xslt
#ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
...
TCP  10.103.56.28:80 rr
...
```

重启了下proxy,发现service ip挂载的还是不对。

```$xslt
# systemctl restart kube-proxy
```

然后看了下这个node节点的状态发现Not Ready

```
#kubectl get node
NAME            STATUS   ROLES    AGE    VERSION
...
10.34.64.129    Not Ready    <none>   69d    v1.16.6
10.34.64.130    Not Ready    <none>   66d    v1.16.6
...
```

因为节点信息是kubelet上报的，所以马上看了下kubelet日志，发现use of closed network connection(https://github.com/kubernetes/kubernetes/issues/87615)
根据issue，kubelet连接异常之后没去重新建立连接，而是复用了之前的tcp连接，导致这个报错，原因是因为使用http2.0来连接存在bug

高赞回答是重启，很运维。。。
```
#!/bin/bash
output=$(journalctl -u kubelet -n 1 | grep "use of closed network connection")
if [[ $? != 0 ]]; then
  echo "Error not found in logs"
elif [[ $output ]]; then
  echo "Restart kubelet"
  systemctl restart kubelet
fi
```

不过还是重启牛逼，重启kubelet，状态恢复业务，然后继续排查出现这个问题原因

```
Jul 27 23:32:07 dc07-prod-k8s-node-bj01host-464129 kubelet[17619]: E0727 23:32:07.831897   17619 kubelet_node_status.go:388] Error updating node status, will retry: error gg
etting node "10.34.64.129": Get https://k8s-api.vdian.net/api/v1/nodes/10.34.64.129?timeout=10s: write tcp 10.34.64.129:13956->10.39.128.21:443: use of closed network connee
ction
Jul 27 23:32:07 dc07-prod-k8s-node-bj01host-464129 kubelet[17619]: E0727 23:32:07.831986   17619 kubelet_node_status.go:388] Error updating node status, will retry: error gg
etting node "10.34.64.129": Get https://k8s-api.vdian.net/api/v1/nodes/10.34.64.129?timeout=10s: write tcp 10.34.64.129:13956->10.39.128.21:443: use of closed network connee
ction
Jul 27 23:32:07 dc07-prod-k8s-node-bj01host-464129 kubelet[17619]: E0727 23:32:07.831999   17619 kubelet_node_status.go:375] Unable to update node status: update node statuu
s exceeds retry count
Jul 27 23:32:08 dc07-prod-k8s-node-bj01host-464129 kubelet[17619]: E0727 23:32:08.825992   17619 reflector.go:123] k8s.io/client-go/informers/factory.go:134: Failed to listt
 *v1beta1.CSIDriver: Get https://k8s-api.vdian.net/apis/storage.k8s.io/v1beta1/csidrivers?limit=500&resourceVersion=0: write tcp 10.34.64.129:13956->10.39.128.21:443: use oo
f closed network connection
Jul 27 23:32:08 dc07-prod-k8s-node-bj01host-464129 kubelet[17619]: E0727 23:32:08.825992   17619 reflector.go:123] k8s.io/client-go/informers/factory.go:134: Failed to listt
 *v1beta1.RuntimeClass: Get https://k8s-api.vdian.net/apis/node.k8s.io/v1beta1/runtimeclasses?limit=500&resourceVersion=0: write tcp 10.34.64.129:13956->10.39.128.21:443: uu
se of closed network connection
Jul 27 23:32:08 dc07-prod-k8s-node-bj01host-464129 kubelet[17619]: E0727 23:32:08.826072   17619 reflector.go:123] k8s.io/kubernetes/pkg/kubelet/kubelet.go:459: Failed to ll
ist *v1.Node: Get https://k8s-api.vdian.net/api/v1/nodes?fieldSelector=metadata.name%3D10.34.64.129&limit=500&resourceVersion=0: write tcp 10.34.64.129:13956->10.39.128.21::
443: use of closed network connection


# systemctl restart kubelet

```

查看nginx日志,时间点和kubelet出异常吻合，说明当时api-server出现异常


```
$ grep k8s-api.vdian.net /data/logs/nginx/cronolog/2020-07-27/2020-07-27-23-access_log|grep '27/Jul/2020:' |awk '$11~50 {print $0}'
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /api/v1/services?limit=500&resourceVersion=0 HTTP/2.0" 502 750 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "12" "api_prod" "0e770000017390e7197e0a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /api/v1/namespaces/default/secrets?fieldSelector=metadata.name%3Dharbor-secret&limit=500&resourceVersion=0 HTTP/2.0" 502 816 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "13" "api_prod" "0e7b0000017390e719830a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /apis/node.k8s.io/v1beta1/runtimeclasses?limit=500&resourceVersion=0 HTTP/2.0" 502 774 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "14" "api_prod" "0e7c0000017390e719840a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /api/v1/pods?fieldSelector=spec.nodeName%3D10.34.64.130&limit=500&resourceVersion=0 HTTP/2.0" 502 793 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "15" "api_prod" "0e7d0000017390e719850a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /api/v1/namespaces/default/secrets?fieldSelector=metadata.name%3Ddefault-token-fbppf&limit=500&resourceVersion=0 HTTP/2.0" 502 822 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "16" "api_prod" "0e7e0000017390e719860a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /apis/storage.k8s.io/v1beta1/csidrivers?limit=500&resourceVersion=0 HTTP/2.0" 502 773 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "17" "api_prod" "0e7f0000017390e719870a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /api/v1/namespaces/kube-system/secrets?fieldSelector=metadata.name%3Ddefault-token-7cjvm&limit=500&resourceVersion=0 HTTP/2.0" 502 826 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "18" "api_prod" "0e800000017390e719880a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:11 +0800] k8s-api.vdian.net "GET /api/v1/nodes?fieldSelector=metadata.name%3D10.34.64.130&limit=500&resourceVersion=0 HTTP/2.0" 502 794 0.000 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 0.000 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "19" "api_prod" "0ebc0000017390e71a450a21827009e0"
10.34.32.147 41300 - - [27/Jul/2020:23:32:12 +0800] k8s-api.vdian.net "PUT /apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/10.34.32.147?timeout=10s HTTP/2.0" 500 199 7.003 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 7.003 "ECDHE-RSA-AES256-GCM-SHA384" "be27127acb7c022c24bec3faf325e27413216370947d10abe4b3b9235a2ec7d1" "." "TLSv1.2" "HTTP/2.0" "" "40" "10.34.98.247:8080" "0f5a0000017390e71c700a21827009e0"
10.34.64.130 61324 - - [27/Jul/2020:23:32:19 +0800] k8s-api.vdian.net "PUT /apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/10.34.64.130?timeout=10s HTTP/2.0" 500 199 7.003 "-" "kubelet/v1.16.6 (linux/amd64) kubernetes/72c3016" "-" 7.003 "ECDHE-RSA-AES256-GCM-SHA384" "be653f51a8893ade65b585c64add88f7a5ab9a53e5fea6058f7e357f8f5aa7a2" "." "TLSv1.2" "HTTP/2.0" "" "28" "10.34.50.114:8080" "187f0000017390e738cb0a21827009e0"
```

查看kube-api日志,3台都出现检查etcd连接失败错误

```
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: I0727 23:30:50.138514   26102 trace.go:116] Trace[535296298]: "GuaranteedUpdate etcd3" type:*core.Endpointt
s (started: 2020-07-27 23:30:43.137151823 +0800 CST m=+4070623.983184304) (total time: 7.001324647s):
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: Trace[535296298]: [7.001324647s] [7.001014676s] END
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: E0727 23:30:50.138575   26102 status.go:71] apiserver received an error that is not an metav1.Status: rpctt
ypes.EtcdError{code:0xe, desc:"etcdserver: request timed out"}
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: I0727 23:30:50.138817   26102 trace.go:116] Trace[861031416]: "Update" url:/api/v1/namespaces/kube-system//
endpoints/kube-scheduler (started: 2020-07-27 23:30:43.136986878 +0800 CST m=+4070623.983019346) (total time: 7.001814071s):
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: Trace[861031416]: [7.001814071s] [7.001696679s] END
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: E0727 23:30:50.264740   26102 status.go:71] apiserver received an error that is not an metav1.Status: rpctt
ypes.EtcdError{code:0xe, desc:"etcdserver: request timed out"}
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: I0727 23:30:50.265539   26102 trace.go:116] Trace[1325373267]: "Get" url:/api/v1/namespaces/ingress-nginx//
services/ingress-nginx (started: 2020-07-27 23:30:43.263279276 +0800 CST m=+4070624.109311738) (total time: 7.002231552s):
Jul 27 23:30:50 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: Trace[1325373267]: [7.002231552s] [7.002213991s] END
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: I0727 23:30:51.538337   26102 healthz.go:191] [+]ping ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]log ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [-]etcd failed: reason withheld
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/generic-apiserver-start-informers ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/start-apiextensions-informers ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/start-apiextensions-controllers ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/crd-informer-synced ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/bootstrap-controller ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/scheduling/bootstrap-system-priority-classes ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/ca-registration ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/start-kube-apiserver-admission-initializer ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/start-kube-aggregator-informers ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/apiservice-registration-controller ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/apiservice-status-available-controller ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/kube-apiserver-autoregistration ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]autoregister-completion ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: [+]poststarthook/apiservice-openapi-controller ok
Jul 27 23:30:51 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: healthz check failed
Jul 27 23:30:53 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: E0727 23:30:53.139709   26102 status.go:71] apiserver received an error that is not an metav1.Status: &errr
ors.errorString{s:"context canceled"}
Jul 27 23:30:53 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: I0727 23:30:53.139958   26102 trace.go:116] Trace[487660225]: "Get" url:/api/v1/namespaces/kube-system/endd
points/kube-scheduler (started: 2020-07-27 23:30:52.140545195 +0800 CST m=+4070632.986577657) (total time: 999.385475ms):
Jul 27 23:30:53 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: Trace[487660225]: [999.385475ms] [999.261535ms] END
Jul 27 23:30:53 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: E0727 23:30:53.654090   26102 status.go:71] apiserver received an error that is not an metav1.Status: &errr
ors.errorString{s:"context canceled"}
Jul 27 23:30:53 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: I0727 23:30:53.654330   26102 trace.go:116] Trace[1631365945]: "Get" url:/api/v1/nodes/10.18.1.80 (startedd
: 2020-07-27 23:30:43.659918755 +0800 CST m=+4070624.505951221) (total time: 9.994367787s):
Jul 27 23:30:53 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: Trace[1631365945]: [9.994367787s] [9.994311229s] END
Jul 27 23:30:54 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: I0727 23:30:54.620410   26102 trace.go:116] Trace[1690954980]: "GuaranteedUpdate etcd3" type:*coordinationn
.Lease (started: 2020-07-27 23:30:47.618879397 +0800 CST m=+4070628.464911867) (total time: 7.001479621s):
Jul 27 23:30:54 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: Trace[1690954980]: [7.001479621s] [7.001221s] END
Jul 27 23:30:54 dc07-prod-k8s-master-bj01host-103450114 kube-apiserver[26102]: E0727 23:30:54.620488   26102 status.go:71] apiserver received an error that is not an metav1.Status: rpctt
ypes.EtcdError{code:0xe, desc:"etcdserver: request timed out"}

```

查看etcd监控发现磁盘被打满了

![image](https://github.com/jinyuchen724/k8s-falldown/raw/master/%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5/%E5%AE%B9%E5%99%A8%E5%90%AF%E5%8A%A8%E7%8A%B6%E6%80%81%E5%8D%A1%E5%9C%A8ContainerCreating/1.png)



为啥etcd磁盘会被打满呢，还是先看etcd日志

```


```
