---
---

Garbage collection is managed by kubelet automatically, mainly including unreferenced
images and dead containers. kubelet applies container garbage collection every minute
and image garbage collection every 5 minutes.
Note that we don't recommend external garbage collection tool generally, since it could
break the behavior of kubelet potentially if it attempts to remove all of the containers
which acts as the tombstone kubelet relies on. Yet those garbage collector aims to deal
with the docker leaking issues would be appreciated.

### Image Collection

kubernetes manages lifecycle of all images through imageManager, with the cooperation
of cadvisor.
The policy for garbage collecting images we apply takes two factors into consideration,
`HighThresholdPercent` and `LowThresholdPercent`. Disk usage above the the high threshold
will trigger garbage collection, which attempts to delete unused images until the low
threshold is met. Least recently used images are deleted first.

### Container Collection

The policy for garbage collecting containers we apply takes on three variables, which can
be user-defined. `MinAge` is the minimum age at which a container can be garbage collected,
zero for no limit. `MaxPerPodContainer` is the max number of dead containers any single
pod (UID, container name) pair is allowed to have, less than zero for no limit.
`MaxContainers` is the max number of total dead containers, less than zero for no limit as well.

kubelet sorts out containers which are unidentified or stay out of bounds set by previous
mentioned three flags. Gernerally the oldest containers are removed first. Since we take both
`MaxPerPodContainer` and `MaxContainers` into consideration, it could happen when they
have conflict -- retaining the max number of containers per pod goes out of range set by max
number of global dead containers. In this case, we would sacrifice the `MaxPerPodContainer`
a little bit. For the worst case, we first downgrade it to 1 container per pod, and then
evict the oldest containers for the greater good.

When kubelet removes the dead containers, all the files inside the container will be cleaned up as well.
Note that we will skip the containers that are not managed by kubelet.

### User Configuration

Users are free to set their own value to address image garbage collection.

1. `image-gc-high-threshold`, the percent of disk usage which triggers image garbage collection.
Default is 90%.
2. `image-gc-low-threshold`, the percent of disk usage to which image garbage collection attempts
to free. Default is 80%.

We also allow users to customize garbage collection policy, basically via following three flags.

1. `minimum-container-ttl-duration`, minimum age for a finished container before it is
garbage collected. Default is 1 minute.
2. `maximum-dead-containers-per-container`, maximum number of old instances to retain
per container. Default is 2.
3. `maximum-dead-containers`, maximum number of old instances of containers to retain globally.
Default is 100.

Note that we highly recommend a large enough value for `maximum-dead-containers-per-container`
to allow at least 2 dead containers retaining per expected container when you customize the flag
configuration. A loose value for `maximum-dead-containers` also assumes importance for a similar reason.
See [this issue](https://github.com/kubernetes/kubernetes/issues/13287) for more details.
