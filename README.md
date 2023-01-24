# secondary-scheduler

Since the default scheduler cannot be modified in ROSA, a secondary kubernetes scheduler may be used.

Deploy a secondary kubernetes scheduler in a custom namespace (this scheduler should be equivalent to the [HighNodeUtilization](https://docs.openshift.com/container-platform/4.12/nodes/scheduling/nodes-scheduler-profiles.html#nodes-scheduler-profiles-about_nodes-scheduler-profiles) scheduler profile)...

```sh
helm upgrade --install secondary-scheduler helm/secondary-scheduler -n secondary-scheduler --create-namespace
```

## test

```sh
export ns=${test_namespace}

# default-scheduler
helm upgrade --install default-scheduler-test-app helm/test-app -n ${ns} --create-namespace

# secondary-scheduler
helm upgrade --install secondary-scheduler-test-app helm/test-app -n ${ns} --set schedulerName=secondary-scheduler --create-namespace

# bad-scheduler
helm upgrade --install bad-test-app helm/test-app -n ${ns} --set schedulerName=bad-scheduler --create-namespace
```

## create machinepools

Add a machine pool test in ROSA with autoscaling enabled and protect the nodes from the default-scheduler using a taint.

Apps using the pool and secondary scheduler need to specify the schedulerName, tolerate the taint and select the nodes.

```sh
export cluster= #Name or ID of the cluster.
rosa create machinepool --cluster=${cluster} \
  --name=test \
  --instance-type=m5.xlarge \
  --enable-autoscaling \
  --min-replicas=0 \
  --max-replicas=2 \
  --labels=descheduler=true,secondaryScheduler=true \
  --taints=secondaryScheduler=true:NoSchedule
```

## test autoscaling

in ROSA, create a machine pool with taint and label. next, update helm/test-app/values-test.yaml with resource requests nodeSelector and tolerations to match the machine pool labels that should autoscale.

```sh
helm upgrade --install secondary-scheduler-test-app helm/test-app -n ${ns} --set schedulerName=secondary-scheduler --set replicaCount=5 -f helm/test-app/values-test.yaml --create-namespace
```

Next, change the resource requests which should cause the pool to scale down nodes.

```sh
helm upgrade --install secondary-scheduler-test-app helm/test-app -n ${ns} --set schedulerName=secondary-scheduler --set replicaCount=5 -f helm/test-app/values-test-small.yaml --create-namespace
```

## delete

```sh
oc delete project test-app
helm delete secondary-scheduler -n secondary-scheduler
oc delete project secondary-scheduler
```
