---
title: "How Kubernetes Deletes A Pod Gracefully"
date: 2020-08-15T16:33:20+09:00
draft: false
toc: true
tags: [
    "kuberentes"
]
categories: [
    "kubernetes", "troubleshooting"
]
---

> This blog will introduce the graceful deletion proceess of pod in source code level. It maybe little bit complicated. If you just want to know the process in high level, you can reference the [official document](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)

## Hey! My pod was stuck in Terminating state

During my on-call time, some users ask for my help with their pods that cannot be deleted successfully in the Kubernetes cluster. Maybe the pod was stuck in the `Terminating` state, maybe "kubectl delete <pod>" has been returned, but that pod object still exists. 

But, the user doesn't always give me a chance to fix it. A "popular" workaround on the Internet tells them "kubectl delete <pod> --force --grace-period=0" can help them. Actually, "force delete" is a kind of dangerous behavior if you didn't know what's a consequence. It may cause the data inconsistency in the Kubernetes cluster. It also breaks the troubling scenario so that we may lose the chance to find an implicit and important issue.

The best approach to troubleshoot this problem is to know the expected deletion behavior first and compare it with the actual behavior, and then fix the difference. So, I want to introduce the graceful deletion process of Kubernetes and related working mechanism on this blog. I hope it can help you to troubleshoot this kind of issue.


## The deletion process

When you sent a request of deleting a pod object to kube-apiserver component by RESTful API or kubectl command, it was handled by `StandardStorage` of kube-apiserver first. Because the primary responsibility of kube-apiserver is to maintain resource data on the etcd cluster, other components like kube-controller-manager and kube-scheduler just watch data changes and move them from the actual state to the desired state.

Here is a `Store` package in kube-apiserver repo; it implemented all methods of [StandardStorage](https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.14.6/-/blob/staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go#L866). The `Delete` function is called.

```go
// Delete removes the item from storage.
func (e *Store) Delete(ctx context.Context, name string, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
    ...
    graceful, pendingGraceful, err := rest.BeforeDelete(e.DeleteStrategy, ctx, obj, options)
    ...
```

### First Round -- from deletion request to kube-apiserver

In `rest.BeforeDelete` function, it invokes `CheckGracefulDelete` method to confirm if the resource you want to delete needs to be deleted gracefully or not. Actually, any resource can implement their graceful deletion logic by `RESTGracefulDeleteStrategy` interface. Now, only native kubernetes Pod resource is supported.

```go
func BeforeDelete(strategy RESTDeleteStrategy, ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) (graceful, gracefulPending bool, err error) {
    ...
	gracefulStrategy, ok := strategy.(RESTGracefulDeleteStrategy)
	if !ok {
		// If we're not deleting gracefully there's no point in updating Generation, as we won't update
		// the obcject before deleting it.
		return false, false, nil
	}
	// if the object is already being deleted, no need to update generation.
	if objectMeta.GetDeletionTimestamp() != nil {
        ...
	}

	if !gracefulStrategy.CheckGracefulDelete(ctx, obj, options) {
        
       return false, false, nil
    }
    ...
}
```

In the `CheckGracefulDelete` function, it tries to calculate the graceful deletion period by different approaches. Those approaches can be divided into three types

```go
// RESTGracefulDeleteStrategy must be implemented by the registry that supports
// graceful deletion.
type RESTGracefulDeleteStrategy interface {
	// CheckGracefulDelete should return true if the object can be gracefully deleted and set
	// any default values on the DeleteOptions.
	CheckGracefulDelete(ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) bool
}

// CheckGracefulDelete allows a pod to be gracefully deleted. It updates the DeleteOptions to
// reflect the desired grace value.
func (podStrategy) CheckGracefulDelete(ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) bool {
	if options == nil {
		return false
	}
	pod := obj.(*api.Pod)
	period := int64(0)
	// user has specified a value
	if options.GracePeriodSeconds != nil {
		period = *options.GracePeriodSeconds
	} else {
		// use the default value if set, or deletes the pod immediately (0)
		if pod.Spec.TerminationGracePeriodSeconds != nil {
			period = *pod.Spec.TerminationGracePeriodSeconds
		}
	}
	// if the pod is not scheduled, delete immediately
	if len(pod.Spec.NodeName) == 0 {
		period = 0
	}
	// if the pod is already terminated, delete immediately
	if pod.Status.Phase == api.PodFailed || pod.Status.Phase == api.PodSucceeded {
		period = 0
	}
	// ensure the options and the pod are in sync
	options.GracePeriodSeconds = &period
	return true
}
```


#### Running Pod

It picks up the `GracePeriodSeconds` of DeleteOptions first. You can specify it by command argument of kubectl or api request. For example, "kubectl delete <pod> --grace-period=10", the `GracePeriodSeconds` should be 30 seconds by default. If you didn't specify it, `pod.Spec.TerminationGracePeriodSeconds` will be used.

#### Pending Pod

The only thing of kube-scheduler did for every pod is update `scheduable node name` to `pod.Spec.NodeName`, so that kubelet on that host can run the pod. So, if `pod.Spec.NodeName` is nil, it means pod doesn't run. It must be fine to delete it at once ( 0 graceful period means delete at once)

#### Terminated Pod

For `Terminated Pod`, it was almost the same as `Pending Pod` because they are not running.

After the graceful deletion period was confirmed, kube-apiserver will set `DeletionTimestamp` and `DeletionGracePeriodSeconds` for the pod.

```go
func BeforeDelete(strategy RESTDeleteStrategy, ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) (graceful, gracefulPending bool, err error) {
    ....
	if !gracefulStrategy.CheckGracefulDelete(ctx, obj, options) {
        
       return false, false, nil
    }
    ...
	now := metav1.NewTime(metav1.Now().Add(time.Second * time.Duration(*options.GracePeriodSeconds)))
	objectMeta.SetDeletionTimestamp(&now)
    objectMeta.SetDeletionGracePeriodSeconds(options.GracePeriodSeconds)
    ...
	return true, false, nil
```

kube-apiserver will update `DeletionTimestamp` and `DeletionGracePeriodSeconds` to the pod by `updateForGracefulDeletionAndFinalizers` function

```go
// Delete removes the item from storage.
func (e *Store) Delete(ctx context.Context, name string, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
    ...
    graceful, pendingGraceful, err := rest.BeforeDelete(e.DeleteStrategy, ctx, obj, options)
    ...
    if graceful || pendingFinalizers || shouldUpdateFinalizers {
		err, ignoreNotFound, deleteImmediately, out, lastExisting = e.updateForGracefulDeletionAndFinalizers(ctx, name, key, options, preconditions, obj)
	}
```

In `updateForGracefulDeletionAndFinalizers` function, kube-apiserver not only updates `DeletionTimestamp` based on graceful deletion time, but also update finalizers to the pod if it found some new finalizers.

`finalizer` is another important concept when a resource object was deleted. It was related to the Gargabe Collector ,cascade-deletion, and ownerReference. If a resource object supports cascade-deletion, it must implement `GarbageCollectionDeleteStrategy` interfaces. Now, you can understand it as a set of `pre-delete` hooks. They are performed one by one and removed after it was finished.

According to the logic of `updateForGracefulDeletionAndFinalizers`, you may know the deletion of the pod was controlled by two guys.

1. finalizers
2. graceful time

Both of them need the resource object implement `GarbageCollectionDeleteStrategy` and `RESTGracefulDeleteStrategy` interfaces. *Only finalizers is nil and graceful time is 0*, that pod can be deleted from the storage successfully. Otherwise it will be blocked.

```go
func (e *Store) updateForGracefulDeletionAndFinalizers(ctx context.Context, name, key string, options *metav1.DeleteOptions, preconditions storage.Preconditions, in runtime.Object) (err error, ignoreNotFound, deleteImmediately bool, out, lastExisting runtime.Object) {
	lastGraceful := int64(0)
	var pendingFinalizers bool
	out = e.NewFunc()
	err = e.Storage.GuaranteedUpdate(
		ctx,
		key,
		out,
		false, /* ignoreNotFound */
		&preconditions,
		storage.SimpleUpdate(func(existing runtime.Object) (runtime.Object, error) {
			graceful, pendingGraceful, err := rest.BeforeDelete(e.DeleteStrategy, ctx, existing, options)
			if err != nil {
				return nil, err
			}
			if pendingGraceful {
				return nil, errAlreadyDeleting
			}

			// Add/remove the orphan finalizer as the options dictates.
			// Note that this occurs after checking pendingGraceufl, so
			// finalizers cannot be updated via DeleteOptions if deletion has
			// started.
			existingAccessor, err := meta.Accessor(existing)
			if err != nil {
				return nil, err
			}
			needsUpdate, newFinalizers := deletionFinalizersForGarbageCollection(ctx, e, existingAccessor, options)
			if needsUpdate {
				existingAccessor.SetFinalizers(newFinalizers)
			}

			lastGraceful = *options.GracePeriodSeconds
			lastExisting = existing
			return existing, nil
		}),
		dryrun.IsDryRun(options.DryRun),
	)
	switch err {
	case nil:
		// If there are pending finalizers, we never delete the object immediately.
		if pendingFinalizers {
			return nil, false, false, out, lastExisting
		}
		if lastGraceful > 0 {
			return nil, false, false, out, lastExisting
        }
        return nil, true, true, out, lastExisting
        ...
}
```

After kube-apiserver updated `DeletionTimestamp` of pod, `deleteImmediately` was returned as false. The first round of the deletion pod was finished. In the next round, kubelet will be a leader.

```go
// Delete removes the item from storage.
func (e *Store) Delete(ctx context.Context, name string, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
	...
	graceful, pendingGraceful, err := rest.BeforeDelete(e.DeleteStrategy, ctx, obj, options)
	if err != nil {
		return nil, false, err
	}
    ....
	if graceful || pendingFinalizers || shouldUpdateFinalizers {
		err, ignoreNotFound, deleteImmediately, out, lastExisting = e.updateForGracefulDeletionAndFinalizers(ctx, name, key, options, preconditions, obj)
	}

	// !deleteImmediately covers all cases where err != nil. We keep both to be future-proof.
	if !deleteImmediately || err != nil {
		return out, false, err
    }
    ...
```

### Second Round -- from kubelet to kube-apiserver

First of all, you need to know. There is a function called [syncLoop](https://sourcegraph.com/github.com/kubernetes/kubernetes@633ab1c/-/blob/pkg/kubelet/kubelet.go#L1726) in the main loop of kubelet. It watched resource changes from multiple places. The most commonplace is kube-apiserver. It will do some reconcile works based on those changes.

```go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.Info("Starting kubelet main sync loop.")
    ...
	for {
        ...
		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}

func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		if !open {
			klog.Errorf("Update channel is closed. Exiting the sync loop.")
			return false
		}

		switch u.Op {
			...
		case kubetypes.DELETE:
			klog.V(2).Infof("SyncLoop (DELETE, %q): %q", u.Source, format.Pods(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
            handler.HandlePodUpdates(u.Pods)
        ....
		}
```

For pod deletion, because of graceful deletion, it was handled as update evnet. Every pod event will be handled by an object called [podWorker](https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.14.6/-/blob/pkg/kubelet/kubelet.go#L2081). It maintains a podUpdate map and invokes `syncPodFn` function to sync the latest change to the "pod". Please pay attention, this pod is not resource object in kube-apiserver, it is a logical pod object on the host where kubelet was deployed.

```go
func (p *podWorkers) managePodLoop(podUpdates <-chan UpdatePodOptions) {
	var lastSyncTime time.Time
	for update := range podUpdates {
		err := func() error {
            podUID := update.Pod.UID
            ...
			err = p.syncPodFn(syncPodOptions{
				mirrorPod:      update.MirrorPod,
				pod:            update.Pod,
				podStatus:      status,
				killPodOptions: update.KillPodOptions,
				updateType:     update.UpdateType,
			})
```

So, in the `syncPodFn` function, the pod is deleted by `kubelet.killpod` function. In the `containerRuntime.KillPod`, it kills containers of the pod in parallel.

```go
func (kl *Kubelet) syncPod(o syncPodOptions) error {
    if !runnable.Admit || pod.DeletionTimestamp != nil || apiPodStatus.Phase == v1.PodFailed {
		var syncErr error
		if err := kl.killPod(pod, nil, podStatus, nil); err != nil {
            ...
        }
    }
}

func (kl *Kubelet) killPod(pod *v1.Pod, runningPod *kubecontainer.Pod, status *kubecontainer.PodStatus, gracePeriodOverride *int64) error {
    ...
	// Call the container runtime KillPod method which stops all running containers of the pod
	if err := kl.containerRuntime.KillPod(pod, p, gracePeriodOverride); err != nil {
		return err
	}
    ...
}
```

When containerRuntimeManager kills containers, it calculates the gracePeriod time first based on `DeletionGracePeriodSeconds` and then tries to stop containers
```go
// killContainer kills a container through the following steps:
// * Run the pre-stop lifecycle hooks (if applicable).
// * Stop the container.
func (m *kubeGenericRuntimeManager) killContainer(pod *v1.Pod, containerID kubecontainer.ContainerID, containerName string, message string, gracePeriodOverride *int64) error {
    ...

	// From this point , pod and container must be non-nil.
	gracePeriod := int64(minimumGracePeriodInSeconds)
	switch {
	case pod.DeletionGracePeriodSeconds != nil:
		gracePeriod = *pod.DeletionGracePeriodSeconds
	case pod.Spec.TerminationGracePeriodSeconds != nil:
		gracePeriod = *pod.Spec.TerminationGracePeriodSeconds
    }
    ...
	err := m.runtimeService.StopContainer(containerID.ID, gracePeriod)
	if err != nil {
		klog.Errorf("Container %q termination failed with gracePeriod %d: %v", containerID.String(), gracePeriod, err)
	} else {
		klog.V(3).Infof("Container %q exited normally", containerID.String())
	}

	m.containerRefManager.ClearRef(containerID)

	return err
}
```

Stop container is easy to do. Kubelet just invokes Stop interface of container runtime. In my environment, the container runtime is Docker. [Docker will send the SIGTERM to cotnainer first, after grace period passed, send the SIGKILL to container finally. ](https://docs.docker.com/engine/reference/commandline/stop/)
```go
func (r *RemoteRuntimeService) StopContainer(containerID string, timeout int64) error {
	// Use timeout + default timeout (2 minutes) as timeout to leave extra time
	// for SIGKILL container and request latency.
	t := r.timeout + time.Duration(timeout)*time.Second
	ctx, cancel := getContextWithTimeout(t)
	defer cancel()

	r.errorMapLock.Lock()
	delete(r.lastError, containerID)
	delete(r.errorPrinted, containerID)
	r.errorMapLock.Unlock()
	_, err := r.runtimeClient.StopContainer(ctx, &runtimeapi.StopContainerRequest{
		ContainerId: containerID,
		Timeout:     timeout,
	})
	if err != nil {
		klog.Errorf("StopContainer %q from runtime service failed: %v", containerID, err)
		return err
	}

	return nil
}
```

After all containers of the pod are cleaned based on the above process, kubelet sends the deletion request of that pod again. It is done by an object called [pod status manager](https://sourcegraph.com/github.com/kubernetes/kubernetes@633ab1c/-/blob/pkg/kubelet/status/status_manager.go#L541). It is responsible for sync pod status from kubelet to kube-apiserver. The only difference with user's deletion request is: grace period is 0. It means kube-apiserver can delete that pod from etcd storage at once.

```go
// syncPod syncs the given status with the API server. The caller must not hold the lock.
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
	if !m.needsUpdate(uid, status) {
		klog.V(1).Infof("Status for pod %q is up-to-date; skipping", uid)
		return
    }
    ...
	if m.canBeDeleted(pod, status.status) {
		deleteOptions := metav1.DeleteOptions{
			GracePeriodSeconds: new(int64),
			// Use the pod UID as the precondition for deletion to prevent deleting a
			// newly created pod with the same name and namespace.
			Preconditions: metav1.NewUIDPreconditions(string(pod.UID)),
		}
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(context.TODO(), pod.Name, deleteOptions)
		if err != nil {
			klog.Warningf("Failed to delete status for pod %q: %v", format.Pod(pod), err)
			return
		}
		klog.V(3).Infof("Pod %q fully terminated and removed from etcd", format.Pod(pod))
		m.deletePodStatus(uid)
    }
```

Now, the second round ended. But the deletion process isn't finished yet. In the next round, the kube-apiserver comes back as the leader.

### Third Round -- from kube-apiserver to etcd

The start point of the third round is from `Delete` method of `StandardStorage` in kube-apiserver. It is almost the same as the first round. In `updateForGracefulDeletionAndFinalizers` function, it will check two points

1. If finalizers of resource object are nil
2. If GracePeriodSeconds is 0

Both requirements are satisfied at the same time, and the `deleteImmediately` will be returned as true. At this time, it will not exit earlier. kube-apiserver deletes the pod from the storage finally.

```go
// Delete removes the item from storage.
func (e *Store) Delete(ctx context.Context, name string, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
    ...
    graceful, pendingGraceful, err := rest.BeforeDelete(e.DeleteStrategy, ctx, obj, options)
    ...
    if graceful || pendingFinalizers || shouldUpdateFinalizers {
		err, ignoreNotFound, deleteImmediately, out, lastExisting = e.updateForGracefulDeletionAndFinalizers(ctx, name, key, options, preconditions, obj)
    }
    
    // first around exits from here
	if !deleteImmediately || err != nil {
		return out, false, err
    }
    
    ...

    // thrid around will delete the pod from storage foever
	klog.V(6).Infof("going to delete %s from registry: ", name)
	out = e.NewFunc()
	if err := e.Storage.Delete(ctx, key, out, &preconditions, dryrun.IsDryRun(options.DryRun)); err != nil {
		// Please refer to the place where we set ignoreNotFound for the reason
		// why we ignore the NotFound error .
		if storage.IsNotFound(err) && ignoreNotFound && lastExisting != nil {
			// The lastExisting object may not be the last state of the object
			// before its deletion, but it's the best approximation.
			out, err := e.finalizeDelete(ctx, lastExisting, true)
			return out, true, err
		}
		return nil, false, storeerr.InterpretDeleteError(err, qualifiedResource, name)
	}
	out, err = e.finalizeDelete(ctx, out, true)
	return out, true, err
```

## About Finalizers

You can see that finalizers of the resource object are also an important factor that can block the deletion process except for graceful time. Actually, finalizers was set when the resource object was created by the controller. When the controller found the DeletionTimestamp of that object wasn't nil, it will process deletion logic based on the finalizers name. Finalizers are just a string slice, which includes the finalizer's name.

In this blog. I will not introduce it too much because it is related to another big topic about Garbage Collection. So I just assume there are no finalizers of finalizers that were removed successfully by default.

## Which reasons can cause the deletion process was stuck

According to the expected behavior of the graceful deletion process, we can know. 

1. If the graceful period is 0 and no finalizers, no one can block the deletion process
2. If the graceful period isn't 0 or some finalizers left on resource object, both of them may block the deletion process


For the finalizers part, there are two possibilities.

1. Native controllers or custom controllers(For Custom Resource) don't process the deletion logic based on finalizers, or it was failed.
2. Deletion logic of finalizers was finished successfully, when controller removes finalizers, it was failed


For the graceful period, the trouble has strong possibilities on kubelet

1. When kubelet invokes container runtime to clean cotnainers, it was stuck. So the podStatus manager still doesn't update GracePeriodSeconds to 0. There are also further reasons, like the volume of pod cannot be unattached or unmounted, or container process was in `Uninterruptible` state. We should investigate in actual trouble scenarios.
2. Kubelet cannot connect to kube-apiserver. This trouble may happen before kubelet receive DELETE event of pod, also may happen after kubelet finished cleaning work. The former case means kubelet doesn't have a chance to do a cleaning worker. The behind case mean kubelet cannot sync the result to kube-apiserver.

## About Force Deletion

As I mentioned at the beginning of this blog, some guys use the workaround like below to delete resource objects.

1. remove finalizers of resource object he(she) wants to delete
2. using `force` and `grace-period=0` argeuments to run kubectl delete command

If you still want to use this workaround to fix the "deletion process was stuck" issue, you must confirm "cleaning work" about the resource you want to delete has been finished. Otherwise, you will encounter further data inconsistency problems. For example, the volume of the previous pod wasn't unmounted successfully, so your new pod cannot be started.

By the way, even if you used `force` and `grace-period=0` argeuments, there is still 2 seconds grace period for that pod. But it's too short.

## Reference

1. https://zhuanlan.zhihu.com/p/161072336 (In Chinese)
2. https://cizixs.com/2017/06/07/kubelet-source-code-analysis-part-2 (In Chinese)
3. https://lwn.net/Articles/288056/
4. https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination