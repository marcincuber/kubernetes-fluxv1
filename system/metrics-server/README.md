# Kubernetes Metrics Server

## User guide

You can find the user guide in
[the official Kubernetes documentation](https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/).

## Design

The detailed design of the project can be found in the following docs:

- [Metrics API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md)
- [Metrics Server](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/metrics-server.md)

For the broader view of monitoring in Kubernetes take a look into
[Monitoring architecture](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/monitoring_architecture.md)

## Manual Deployment

In order to deploy metrics-server run the following from current directory:

```console
$ kubectl apply -f .
```

## Flags

Metrics Server supports all the standard Kubernetes API server flags, as
well as the standard Kubernetes `glog` logging flags.  The most
commonly-used ones are:

- `--logtostderr`: log to standard error instead of files in the
  container.  You generally want this on.

- `--v=<X>`: set log verbosity.  It's generally a good idea to run a log
  level 1 or 2 unless you're encountering errors.  At log level 10, large
  amounts of diagnostic information will be reported, include API request
  and response bodies, and raw metric results from Kubelet.

- `--secure-port=<port>`: set the secure port.  If you're not running as
  root, you'll want to set this to something other than the default (port
  443).

- `--tls-cert-file`, `--tls-private-key-file`: the serving certificate and
  key files.  If not specified, self-signed certificates will be
  generated, but it's recommended that you use non-self-signed
  certificates in production.

Additionally, Metrics Server defines a number of flags for configuring its
behavior:

- `--metric-resolution=<duration>`: the interval at which metrics will be
  scraped from Kubelets (defaults to 60s).

- `--kubelet-insecure-tls`: skip verifying Kubelet CA certificates.  Not
  recommended for production usage, but can be useful in test clusters
  with self-signed Kubelet serving certificates.

- `--kubelet-port`: the port to use to connect to the Kubelet (defaults to
  the default secure Kubelet port, 10250).

- `--kubelet-preferred-address-types`: the order in which to consider
  different Kubelet node address types when connecting to Kubelet.
  Functions similarly to the flag of the same name on the API server.

## Autoscaling Algorithm

The autoscaler is implemented as a control loop. It periodically queries pods
described by `Status.PodSelector` of Scale subresource, and collects their CPU
utilization. Then, it compares the arithmetic mean of the pods' CPU utilization
with the target defined in `Spec.CPUUtilization`, and adjusts the replicas of
the Scale if needed to match the target (preserving condition: MinReplicas <=
Replicas <= MaxReplicas).

The period of the autoscaler is controlled by the
`--horizontal-pod-autoscaler-sync-period` flag of controller manager. The
default value is 30 seconds.


CPU utilization is the recent CPU usage of a pod (average across the last 1
minute) divided by the CPU requested by the pod. In Kubernetes version 1.1, CPU
usage is taken directly from Heapster.

The target number of pods is calculated from the following formula:

```
TargetNumOfPods = ceil(sum(CurrentPodsCPUUtilization) / Target)
```

Starting and stopping pods may introduce noise to the metric (for instance,
starting may temporarily increase CPU). So, after each action, the autoscaler
should wait some time for reliable data. Scale-up can only happen if there was
no rescaling within the last 3 minutes. Scale-down will wait for 5 minutes from
the last rescaling. Moreover any scaling will only be made if:
`avg(CurrentPodsConsumption) / Target` drops below 0.9 or increases above 1.1
(10% tolerance). Such approach has two benefits:

* Autoscaler works in a conservative way. If new user load appears, it is
important for us to rapidly increase the number of pods, so that user requests
will not be rejected. Lowering the number of pods is not that urgent.

* Autoscaler avoids thrashing, i.e.: prevents rapid execution of conflicting
decision if the load is not stable.

## Relative vs. absolute metrics

We chose values of the target metric to be relative (e.g. 90% of requested CPU
resource) rather than absolute (e.g. 0.6 core) for the following reason. If we
choose absolute metric, user will need to guarantee that the target is lower
than the request. Otherwise, overloaded pods may not be able to consume more
than the autoscaler's absolute target utilization, thereby preventing the
autoscaler from seeing high enough utilization to trigger it to scale up. This
may be especially troublesome when user changes requested resources for a pod
because they would need to also change the autoscaler utilization threshold.
Therefore, we decided to choose relative metric. For user, it is enough to set
it to a value smaller than 100%, and further changes of requested resources will
not invalidate it.
