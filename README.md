# kubernetes-etcd-cluster

Based on https://github.com/helm/charts/tree/master/incubator/etcd without helm, because helm sort of sucks :)
I added a few bits like actual PVC, auto scalar and a different image version, not much more than that.
_Just works_

## Dependencies

- Golang 1.9.X or above  
- `go get github.com/AlexsJones/vortex`

## deployment
```
./build_environment.sh default
kubectl exec etcd-0 etcdctl cluster-health -n etcd
```


## quick test

```
kubectl exec etcd-0 -n etcd -- etcdctl set foo bar
kubectl exec etcd-2 -n etcd -- etcdctl get foo
```

## Failover
If any etcd member fails it gets re-joined eventually. You can test the scenario by killing process of one of the replicas:
```
$ ps aux | grep etcd-1
$ kill -9 ETCD_1_PID
$ kubectl get pods -l "app=etcd"
NAME                 READY     STATUS        RESTARTS   AGE
etcd-0               1/1       Running       0          54s
etcd-2               1/1       Running       0          51s
```
After a while:
```
$ kubectl get pods -l "app=etcd"
NAME                 READY     STATUS    RESTARTS   AGE
etcd-0               1/1       Running   0          1m
etcd-1               1/1       Running   0          20s
etcd-2               1/1       Running   0          1m
```
You can check state of re-joining from etcd-1's logs:
```
$ kubectl logs etcd-1
Waiting for etcd-0.etcd to come up
Waiting for etcd-1.etcd to come up
ping: bad address 'etcd-1.etcd'
Waiting for etcd-1.etcd to come up
Waiting for etcd-2.etcd to come up
Re-joining etcd member
Updated member with ID 7fd61f3f79d97779 in cluster
2016-06-20 11:04:14.962169 I | etcdmain: etcd Version: 2.2.5
2016-06-20 11:04:14.962287 I | etcdmain: Git SHA: bc9ddf2
```
...
Scaling using kubectl
This is for reference. Scaling should be managed by helm upgrade

The etcd cluster can be scale up by running kubectl patch or kubectl edit. For instance,
```
$ kubectl get pods -l "app=etcd"
NAME      READY     STATUS    RESTARTS   AGE
etcd-0    1/1       Running   0          7m
etcd-1    1/1       Running   0          7m
etcd-2    1/1       Running   0          6m
```
```
$ kubectl patch statefulset/etcd -p '{"spec":{"replicas": 5}}'
"etcd" patched
```
```
$ kubectl get pods -l "app=etcd"
NAME      READY     STATUS    RESTARTS   AGE
etcd-0    1/1       Running   0          8m
etcd-1    1/1       Running   0          8m
etcd-2    1/1       Running   0          8m
etcd-3    1/1       Running   0          4s
etcd-4    1/1       Running   0          1s
Scaling-down is similar. For instance, changing the number of replicas to 4:
```
```
$ kubectl edit statefulset/etcd
statefulset "etcd" edited
```
$ kubectl get pods -l "app=etcd"
NAME      READY     STATUS    RESTARTS   AGE
etcd-0    1/1       Running   0          8m
etcd-1    1/1       Running   0          8m
etcd-2    1/1       Running   0          8m
etcd-3    1/1       Running   0          4s
```

Once a replica is terminated (either by running kubectl delete pod etcd-ID or scaling down), content of /var/run/etcd/ directory is cleaned up. If any of the etcd pods restarts (e.g. caused by etcd failure or any other), the directory is kept untouched so the pod can recover from the failure.
