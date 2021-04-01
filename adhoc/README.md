Use on clusters that don't have the newest RP rolled out that updates routefix DS to setup iptables rules to drop ICMP packets.

Will have to be cleaned up after RP rollout.

# Deploy

1. Deploy the DS

```shell
kubectl create -f 00_privileged-routefix-adhoc.scc.yaml
kubectl create -f 06_routefix-adhoc.bindings.yaml
kubectl create -f 10_add-iptables.cm.yaml
kubectl create -f 20_routefix-adhoc.ds.yaml
```

2. Make note of the cluster this is applied on.  We need to clean this up **before** RP rollout.  The detect container logs with "adhoc=true" in the output, this can be used to search for logs in Geneva.  This is not set in the RP based DS.
3. Mitigate the issues on the cluster (restart oauth pods or reboot impacted nodes)

# Validate

Each of the pods has 2 containers.  You can verify both.

## drop-icmp
Tail one of the pods.  expect to see 

```shell
POD=$(oc -n openshift-azure-routefix get pods -l app=routefix-adhoc | grep Running | awk '{print $1}' | head -n1)
oc -n openshift-azure-routefix logs $POD -c drop-icmp -f
```

Expect to see something like this (note the `Bad rule` bit is expected, subsequent executions will not show this):

```
++ date '+%m%d %H:%M:%S.%N'
+ echo 'I0401 15:46:50.609186391 - drop-icmp - start drop-icmp aro-v4-shared-gxqb4-master-1'
+ iptables -X CHECK_ICMP_SOURCE
I0401 15:46:50.609186391 - drop-icmp - start drop-icmp aro-v4-shared-gxqb4-master-1
iptables v1.8.4 (nf_tables):  CHAIN_USER_DEL failed (Device or resource busy): chain CHECK_ICMP_SOURCE
+ true
+ iptables -N CHECK_ICMP_SOURCE
iptables: Chain already exists.
+ true
+ iptables -F CHECK_ICMP_SOURCE
+ iptables -D INPUT -p icmp --icmp-type fragmentation-needed -j CHECK_ICMP_SOURCE
+ iptables -I INPUT -p icmp --icmp-type fragmentation-needed -j CHECK_ICMP_SOURCE
+ iptables -N ICMP_ACTION
iptables: Chain already exists.
+ true
+ iptables -F ICMP_ACTION
+ iptables -A ICMP_ACTION -j LOG
+ iptables -A ICMP_ACTION -j DROP
+ oc observe nodes -a '{ .status.addresses[1].address }' -- /tmp/add_iptables.sh
Flag --argument has been deprecated, and will be removed in a future release. Use --template instead.
# 2021-04-01T15:46:51Z Sync started
# 2021-04-01T15:46:51Z Sync 135850556   /tmp/add_iptables.sh aro-v4-shared-gxqb4-worker-eastus1-xh9ml 10.0.2.4
Adding ICMP drop rule for '10.0.2.4' 
iptables: Bad rule (does a matching rule exist in that chain?).
# 2021-04-01T15:46:51Z Sync 135852562   /tmp/add_iptables.sh aro-v4-shared-gxqb4-worker-eastus2-rknnp 10.0.2.8
Adding ICMP drop rule for '10.0.2.8' 
iptables: Bad rule (does a matching rule exist in that chain?).
# 2021-04-01T15:46:51Z Sync 135853614   /tmp/add_iptables.sh aro-v4-shared-gxqb4-worker-eastus3-nw657 10.0.2.6
Adding ICMP drop rule for '10.0.2.6' 
iptables: Bad rule (does a matching rule exist in that chain?).
# 2021-04-01T15:46:51Z Sync 135852905   /tmp/add_iptables.sh aro-v4-shared-gxqb4-infra-eastus1-28wfk 10.0.2.9
Adding ICMP drop rule for '10.0.2.9' 
iptables: Bad rule (does a matching rule exist in that chain?).
# 2021-04-01T15:46:51Z Sync 135852582   /tmp/add_iptables.sh aro-v4-shared-gxqb4-master-0 10.0.0.7
Adding ICMP drop rule for '10.0.0.7' 
iptables: Bad rule (does a matching rule exist in that chain?).
# 2021-04-01T15:46:51Z Sync 135853322   /tmp/add_iptables.sh aro-v4-shared-gxqb4-master-1 10.0.0.5
Adding ICMP drop rule for '10.0.0.5' 
iptables: Bad rule (does a matching rule exist in that chain?).
# 2021-04-01T15:46:51Z Sync 135851603   /tmp/add_iptables.sh aro-v4-shared-gxqb4-master-2 10.0.0.8
Adding ICMP drop rule for '10.0.0.8' 
...
```

## detect

Each pod logs status.  Just look for last log line to see that it's getting something.

```shell
for POD in $(oc -n openshift-azure-routefix get pods -l app=routefix-adhoc | grep Running | awk '{print $1}');
do
    echo -n "$POD: "
    oc -n openshift-azure-routefix logs $POD -c detect | tail -n1
done
```
Expect something like this (note you might see `broken=true` which means the node has a problem):

```
routefix-adhoc-2q9f8: 2021-04-01 15:49:50 table=10 actions=drop packets=1 broken=false adhoc=true
routefix-adhoc-675qh: 2021-04-01 15:49:50 table=10 actions=drop packets=1 broken=false adhoc=true
routefix-adhoc-fbfck: 2021-04-01 15:49:50 table=10 actions=drop packets=0 broken=false adhoc=true
routefix-adhoc-k467s: 2021-04-01 15:49:51 table=10 actions=drop packets=1 broken=false adhoc=true
routefix-adhoc-m75m7: 2021-04-01 15:49:50 table=10 actions=drop packets=1 broken=false adhoc=true
routefix-adhoc-vb9ks: 2021-04-01 15:49:50 table=10 actions=drop packets=0 broken=false adhoc=true
routefix-adhoc-xl4zx: 2021-04-01 15:49:50 table=10 actions=drop packets=1 broken=false adhoc=true
```

# Cleanup

WARNING must be cleaned up before RP rollout, as drop-icmp container doesn't work from multiple containers.

NOTE do **not** delete the ConfigMap.
NOTE deleting the DS leaves the iptables rules in place.

1. Delete the adhoc DS and binding.

```shell
kubectl delete -f 00_privileged-routefix-adhoc.scc.yaml
kubectl delete -f 06_routefix-adhoc.bindings.yaml
kubectl delete -f 20_routefix-adhoc.ds.yaml
```
# Troubleshoot

If the "real" `routefix` DS gets into a bad state for any reason it is safe to delete it.  ARO operator will redeploy it and all its dependencies.

```shell
kubectl -n openshift-azure-routefix delete daemonset.apps/routefix
```
