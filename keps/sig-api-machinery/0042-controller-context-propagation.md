---
title: Controller Context Propagation
authors:
- "@dashpole"
- "@logicalhan"
owning-sig: sig-api-machinery
participating-sigs:
- sig-instrumentation
approvers:
-
creation-date: 2020-04-29
last-updated: 2020-04-29
status: provisional
---

# Controller Context Propagation

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Design Details](#design-details)
<!-- /toc -->

## Summary

This proposal can be construed as an extension of the work started in the [client-go ctx KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20200123-client-go-ctx.md). We currently do not plumb context properly in our controllers, so we actually don't propagate anything just yet. This KEP proposes a minimal set of changes such that we can start propogating context changes in our controllers.

## Motivation

Having contextual information about why controllers are doing things is nice, since may be helpful to figure out what a controller is doing and why it is doing it. In the [client-go ctx KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20200123-client-go-ctx.md), we introduced context propogation in client-go; this KEP is motivated by the desire to leverage those changes so we can utilize the new functionality.

### Goals

- Plumb context through the main controller loops.
- Establish a wrapping pattern such that we can centralize the logic what we intend to serialize and propagate in context objects.

### Non-Goals

- We do not intend to establish exactly what gets serialized into the context object.

## Proposal

The recommended approach is initialize a context from the beginning of a controller action and pass that to controller client-go invocations.

## Design Details

This is an example change to `Pods.Get` implementation:

```go
func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
  diff := len(filteredPods) - int(*(rs.Spec.Replicas))
  // ...
  if diff < 0 {
    wrappedctx := contextutils.StartControllerContext(context.Background(), rs, "replicaset.CreatePod")
    defer wrappedctx.endCtx()
    err = rsc.podControl.CreatePodsWithControllerRef(wrappedctx.GetCtx(), rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
    if err != nil && errors.IsTimeout(err) {
      return nil
    }
    return err
  }
  // ... more replicaset controller stuff
}
```

