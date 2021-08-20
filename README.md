Containerd logging test
=======================

Test logging output changes from Docker/Moby to Containerd runtimes by deploying sample app on two node pools (1.19.x and 1.18.x):

```sh
kubectl get nodes -o wide
```

Output:

```
NAME                               ...   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
...
aks-nodepool1-19079225-vmss000005  ...   v1.20.7    10.1.0.66     <none>        Ubuntu 18.04.5 LTS   5.4.0-1047-azure   containerd://1.4.4+azure
aks-nodepool2-19079225-vmss000000  ...   v1.18.19   10.1.0.97     <none>        Ubuntu 18.04.5 LTS   5.4.0-1055-azure   docker://19.3.14
```

Deploy sample app (azure-voting - updated to log to STDOUT), generate log activity by using the app:

```sh
kubectl create ns app1
kubectl apply -f azure-vote-nodepool1.yaml -n app1 # nodeSelector => "agentpool": nodepool1
kubectl get all -n app1

kubectl create ns app2
kubectl apply -f azure-vote-nodepool2.yaml -n app2 # nodeSelector => "agentpool": nodepool2
kubectl get all -n app2

kubectl port-forward svc/azure-vote-front 8080:80 -n app1
# Vote a few times, checl container logs and schema in Log Analytics
# CTRL+C

kubectl port-forward svc/azure-vote-front 8081:80 -n app2
# Vote a few times, checl container logs and schema in Log Analytics
# CTRL+C
```

Containerd logging checks
-------------------------

In Log Analytics (KQL):

```kql
ContainerLog
| where LogEntry contains "voting_app"
| where Computer contains "nodepool1"
```

Output line:

```
20/08/2021, 2:54:50.148 am	aks-nodepool1-19079225-vmss000005	20/08/2021, 2:55:28.000 am	02646d855b4d6ee8f257de047366d0f16738241f26fcf8d2af4827e573c8ab6e					2021-08-20 02:54:50,148 - voting_app - DEBUG - VOTE2VALUE = Dogs	stderr	ContainerLog	/subscriptions/<redacted>/resourcegroups/aks-demos/providers/microsoft.containerservice/managedclusters/aks-cni	<redacted-tenant-id>	Containers	
```

Fields schema:

```
TenantId
<redacted-tenant-id>

SourceSystem
Containers

TimeGenerated [UTC]
2021-08-20T02:54:50.148Z

Computer
aks-nodepool1-19079225-vmss000005

TimeOfCommand [UTC]
2021-08-20T02:55:28Z

ContainerID
02646d855b4d6ee8f257de047366d0f16738241f26fcf8d2af4827e573c8ab6e

LogEntry
2021-08-20 02:54:50,148 - voting_app - DEBUG - VOTE2VALUE = Dogs

LogEntrySource
stderr

Type
ContainerLog

_ResourceId
/subscriptions/<redacted>/resourcegroups/aks-demos/providers/microsoft.containerservice/managedclusters/aks-cni
```

SSH onto the AKS node (1.19.x) and check log format:

```sh
kubectl ssh-jump aks-nodepool1-19079225-vmss000005

sudo crictl logs 02646d855b4d6
```

`crictl logs` output:

```
[pid: 14|app: 0|req: 31/45] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 04:30:09 2021] POST / => generated 952 bytes in 2 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
127.0.0.1 - - [20/Aug/2021:04:30:09 +0000] "POST / HTTP/1.1" 200 952 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73" "-"
2021-08-20 04:30:09,633 - voting_app - DEBUG - Saving votes
2021-08-20 04:30:09,634 - voting_app - DEBUG - vote1 = 14
2021-08-20 04:30:09,634 - voting_app - DEBUG - vote2 = 22
[pid: 14|app: 0|req: 32/46] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 04:30:09 2021] POST / => generated 952 bytes in 1 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
127.0.0.1 - - [20/Aug/2021:04:30:09 +0000] "POST / HTTP/1.1" 200 952 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73" "-"
```

Check the raw log file:

```sh
cd /var/log/containers

sudo tail -f azure-vote-front-77dd65d48f-6f65k_app1_azure-vote-front-02646d855b4d6ee8f257de047366d0f16738241f26fcf8d2af4827e573c8ab6e.log
```

CRI log format output:

```
2021-08-20T05:32:43.575962485Z stderr F 2021-08-20 05:32:43,575 - voting_app - DEBUG - Saving votes
2021-08-20T05:32:43.578258765Z stdout F 127.0.0.1 - - [20/Aug/2021:05:32:43 +0000] "POST / HTTP/1.1" 200 952 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73" "-"
2021-08-20T05:32:43.578358864Z stderr F 2021-08-20 05:32:43,577 - voting_app - DEBUG - vote1 = 17
2021-08-20T05:32:43.578366964Z stderr F 2021-08-20 05:32:43,577 - voting_app - DEBUG - vote2 = 24
2021-08-20T05:32:43.578372464Z stderr F [pid: 14|app: 0|req: 37/51] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 05:32:43 2021] POST / => generated 952 bytes in 3 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
2021-08-20T05:32:44.829298056Z stderr F 2021-08-20 05:32:44,828 - voting_app - DEBUG - Saving votes
2021-08-20T05:32:44.830126349Z stderr F 2021-08-20 05:32:44,829 - voting_app - DEBUG - vote1 = 17
2021-08-20T05:32:44.830522246Z stderr F 2021-08-20 05:32:44,829 - voting_app - DEBUG - vote2 = 25
2021-08-20T05:32:44.830547546Z stdout F 127.0.0.1 - - [20/Aug/2021:05:32:44 +0000] "POST / HTTP/1.1" 200 952 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73" "-"
2021-08-20T05:32:44.830660645Z stderr F [pid: 13|app: 0|req: 15/52] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 05:32:44 2021] POST / => generated 952 bytes in 1 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
```

Here you can see the CRI log format is a text line.

Docker/Moby logging checks
--------------------------

In Log Analytics (KQL):

```kql
ContainerLog
| where LogEntry contains "voting_app"
| where Computer contains "nodepool2"
```

Output line:

```
20/08/2021, 3:02:44.716 am	aks-nodepool2-19079225-vmss000000	20/08/2021, 3:02:50.000 am	fb39c8cb966184a6a0c9949f4cb230bd3233f078f3e50d6000bf7d7110298cbf					2021-08-20 03:02:44,715 - voting_app - DEBUG - vote1 = 8	stderr	ContainerLog	/subscriptions/<redacted>/resourcegroups/aks-demos/providers/microsoft.containerservice/managedclusters/aks-cni	<redacted-tenant-id>	Containers	
```

Fields schema:

```
TenantId
<redacted-tenant-id>

SourceSystem
Containers

TimeGenerated [UTC]
2021-08-20T03:02:44.716Z

Computer
aks-nodepool2-19079225-vmss000000

TimeOfCommand [UTC]
2021-08-20T03:02:50Z

ContainerID
fb39c8cb966184a6a0c9949f4cb230bd3233f078f3e50d6000bf7d7110298cbf

LogEntry
2021-08-20 03:02:44,715 - voting_app - DEBUG - vote1 = 8 

LogEntrySource
stderr

Type
ContainerLog

_ResourceId
/subscriptions/<redacted>/resourcegroups/aks-demos/providers/microsoft.containerservice/managedclusters/aks-cni
```

SSH onto the AKS node (1.18.x) and check log format:

```sh
kubectl ssh-jump aks-nodepool2-19079225-vmss000000

docker logs -f fb39c8cb9661
```

`docker logs` output:

```
[pid: 13|app: 0|req: 23/59] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 04:32:27 2021] POST / => generated 951 bytes in 3 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
127.0.0.1 - - [20/Aug/2021:04:32:27 +0000] "POST / HTTP/1.1" 200 951 "http://localhost:8081/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73" "-"
2021-08-20 04:32:27,917 - voting_app - DEBUG - Saving votes
2021-08-20 04:32:27,918 - voting_app - DEBUG - vote1 = 18
2021-08-20 04:32:27,918 - voting_app - DEBUG - vote2 = 6
[pid: 14|app: 0|req: 37/60] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 04:32:27 2021] POST / => generated 951 bytes in 2 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
127.0.0.1 - - [20/Aug/2021:04:32:27 +0000] "POST / HTTP/1.1" 200 951 "http://localhost:8081/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73" "-"
2021-08-20 04:32:29,277 - voting_app - DEBUG - Saving votes
2021-08-20 04:32:29,279 - voting_app - DEBUG - vote1 = 18
2021-08-20 04:32:29,279 - voting_app - DEBUG - vote2 = 7
[pid: 14|app: 0|req: 38/61] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 04:32:29 2021] POST / => generated 951 bytes in 2 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)
127.0.0.1 - - [20/Aug/2021:04:32:29 +0000] "POST / HTTP/1.1" 200 951 "http://localhost:8081/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73" "-"

Check the raw log file:

```sh
cd /var/log/containers

sudo tail -f azure-vote-front-5d8f69dffb-65wbp_app2_azure-vote-front-fb39c8cb966184a6a0c9949f4cb230bd3233f078f3e50d6000bf7d7110298cbf.log
```

Docker JSON log format output:

```json
{"log":"2021-08-20 05:29:57,094 - voting_app - DEBUG - Saving votes\n","stream":"stderr","time":"2021-08-20T05:29:57.095261159Z"}
{"log":"2021-08-20 05:29:57,095 - voting_app - DEBUG - vote1 = 21\n","stream":"stderr","time":"2021-08-20T05:29:57.096089734Z"}
{"log":"2021-08-20 05:29:57,095 - voting_app - DEBUG - vote2 = 8\n","stream":"stderr","time":"2021-08-20T05:29:57.096459968Z"}
{"log":"[pid: 14|app: 0|req: 41/65] 127.0.0.1 () {64 vars in 1231 bytes} [Fri Aug 20 05:29:57 2021] POST / =\u003e generated 951 bytes in 2 msecs (HTTP/1.1 200) 2 headers in 80 bytes (1 switches on core 0)\n","stream":"stderr","time":"2021-08-20T05:29:57.096798398Z"}
{"log":"127.0.0.1 - - [20/Aug/2021:05:29:57 +0000] \"POST / HTTP/1.1\" 200 951 \"http://localhost:8081/\" \"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73\" \"-\"\n","stream":"stdout","time":"2021-08-20T05:29:57.09703572Z"}
```

Conclusion
----------

* Log Analytics KQL container logs and fields appear identical for docker/moby and containerd logs
* `docker logs` and `crictl logs` container logs and fields appear identical for docker/moby and containerd logs
* `/var/log/containers` raw container logs are **different** for docker/moby and cri log formats
* There should be no impact to your log handling or queries when upgrading from 1.18 (docker/moby) to 1.19+ (containerd) provided you do not access the raw container logs directly.  Some 3pp log forwarders, like fluentbit do this so please be aware of changes.  Azure Monitor's container insights handles both formats (via the omsagent daemonset).
