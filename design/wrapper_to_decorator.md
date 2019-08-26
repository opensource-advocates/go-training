- Let us analyse this wrapper logic

```go
// WaitForCacheSync is a wrapper around cache.WaitForCacheSync that
// generates log messages indicating that the controller identified
// by controllerName is waiting for syncs, followed by either a
// successful or failed sync.
func WaitForCacheSync(
	controllerName string,
	stopCh <-chan struct{},
	cacheSyncs ...cache.InformerSynced,
) bool {
	glog.Infof("Waiting for caches to sync for controller %q", controllerName)

	if !cache.WaitForCacheSync(stopCh, cacheSyncs...) {
		utilruntime.HandleError(fmt.Errorf(
			"Unable to sync caches for controller %q", controllerName,
		))
		return false
	}

	glog.Infof("Caches are synced for controller %q", controllerName)
	return true
}
```

### Observations:
- Is the code readable?
  - :+1: This is quite readable
- is the code maintainable?
  - :+1: Logic seems frozen & seems no maintenance
  - :nerd_face: Note it **seems** as frozen, simple, etc. It may or may not remain such forever.
- Can the code be unit tested?
  - :no_entry_sign: It needs to mock `cache` as well as `utilruntime` packages
- Does this code follow Single Responsibility Pattern?
  - :no_entry_sign: It has logic that goes beyond log wrapper boundaries
  - :confused: It treats un-successful sync as an error

#### Suggestions
- In this block we try functional approach to manage decorative pieces of logic
```go
// WaitForCacheSyncFn is a typed function that adheres to
// cache.WaitForCacheSync signature
type WaitForCacheSyncFn func(
	stop <-chan struct{}, isSyncFns ...cache.InformerSynced,
) bool
```

```go
// CacheSyncTimeTaken is a decorator around
// WaitForCacheSyncFn that logs the time taken for
// all caches to sync for the given controller
func WaitForCacheSyncTimeTaken(
	controllerName string,
	fn WaitForCacheSyncFn,
) WaitForCacheSyncFn {
	return func(stop <-chan struct{}, isSyncFns ...cache.InformerSynced) bool {
		start := time.Now()
		defer glog.Infof(
			"Controller %s cache sync took %s",
			controllerName,
			time.Now().Sub(start),
		)
		return fn(stop, isSyncFns...)
	}
}
```
```go
// CacheSyncFailureAsError is a decorator around
// WaitForCacheSyncFn that logs an error if all caches could
// not be sync-ed for the given controller
func CacheSyncFailureAsError(
	controllerName string,
	fn WaitForCacheSyncFn,
) WaitForCacheSyncFn {
	return func(stop <-chan struct{}, isSyncFns ...cache.InformerSynced) bool {
		synced := fn(stop, isSyncFns...)
		if !synced {
			utilruntime.HandleError(fmt.Errorf(
				"Unable to sync caches for controller %s", 
                                controllerName,
			))
		}
		return synced
	}
}
```
- this is how above is used/invoked
```go
// decorate WaitForCacheSync with time taken and logging logic
waitForCacheSync := k8s.CacheSyncTimeTaken(
  controllerName,
  k8s.CacheSyncFailureAsError(
    controllerName,
    cache.WaitForCacheSync,
  ),
)

if !waitForCacheSync(pc.stopCh, syncFuncs...) {
  // We wait forever unless Stop() is called, so this isn't an error.
  glog.Warningf(
    "CompositeController %s cache sync never finished", controllerName,
  )
return
}
```
