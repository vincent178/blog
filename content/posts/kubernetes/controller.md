---
title: "Controller"
date: 2020-04-25T05:05:58+08:00
draft: true
---

```go
// 默认的 clientset
kubeClient, err := kubernetes.NewForConfig(cfg)

// 自定义资源的 clientset
// 通过 client-gen 自动生成的
// 代码在 pkg/generated/clientset/versioned/
exampleClient, err := clientset.NewForConfig(cfg)

// 新建默认的 informer factory
kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)

// 新建自定义资源的 informer factory
// informer factory 中的 informers 是懒加载的
exampleInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)

// 新建 controller
controller := NewController(kubeClient, exampleClient,
    kubeInformerFactory.Apps().V1().Deployments(),
    exampleInformerFactory.Samplecontroller().V1alpha1().Foos())

// 启动默认的 informer factory
kubeInformerFactory.Start(stopCh)

// 启动自定义资源的 informer factory
exampleInformerFactory.Start(stopCh)

// 启动 controller
controller.Run(2, stopCh);
```

`exampleInformerFactory.Samplecontroller().V1alpha1().Foos()` 返回的是由 `*v1alpha1.fooInformer` 实现的 v1alpha1.FooInformer 接口。

调用链
-> factory.NewSharedInformerFactory() SharedInformerFactory
-> factory.NewSharedInformerFactoryWithOptions() SharedInformerFactory (实例是 sharedInformerFactory)
-> SharedInformerFactory.Samplecontroller() samplecontroller.Interface
-> samplecontroller.New() samplecontroller.Interface (实例是 samplecontroller.group)
-> samplecontroller.Interface.V1alpha1() v1alpha1.Interface
-> v1alpha1.New() v1alpha1.Interface (实例是 v1alpha1.version)
-> v1alpha1.Interface.Foos() FooInformer (实例是 v1alpha1.fooInformer)

NewController 中声明了需要监听的事件

```go
func NewController(
    kubeclientset kubernetes.Interface,
    sampleclientset clientset.Interface,
    deploymentInformer appsinformers.DeploymentInformer,
    fooInformer informers.FooInformer) *Controller {
    // ...
    fooInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueueFoo,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueueFoo(new)
        },
    })
    // ...
    return controller
}
```

前面提到了 informer factory 中的 informers 是懒加载，在首次调用 InformerFor 的时候，如果找不到会实例化一个并且缓存下来。

```go
func (f *fooInformer) Informer() cache.SharedIndexInformer {
    return f.factory.InformerFor(&samplecontrollerv1alpha1.Foo{}, f.defaultInformer)
}

func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cach    e.SharedIndexInformer {
    f.lock.Lock()
    defer f.lock.Unlock()

    informerType := reflect.TypeOf(obj)
    informer, exists := f.informers[informerType]
    if exists {
        return informer
    }

    // ...

    informer = newFunc(f.client, resyncPeriod)
    f.informers[informerType] = informer

    return informer
}
```

而这里的 defaultInformer 就是调用 `NewFilteredFooInformer`
```
func NewFilteredFooInformer(client versioned.Interface, namespace string, resyncPeriod time.Duration, indexers c    ache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
    return cache.NewSharedIndexInformer(
        &cache.ListWatch{
            ListFunc: func(options v1.ListOptions) (runtime.Object, error) {
                return client.SamplecontrollerV1alpha1().Foos(namespace).List(context.TODO(), options)
            },
            WatchFunc: func(options v1.ListOptions) (watch.Interface, error) {
                return client.SamplecontrollerV1alpha1().Foos(namespace).Watch(context.TODO(), options)
            },
        },
        &samplecontrollerv1alpha1.Foo{},
        resyncPeriod,
        indexers,
    )
}
```

`cache.NewSharedIndexInformer` 返回的是 sharedIndexInformer
```go
type sharedIndexInformer struct {
    indexer    Indexer
    controller Controller

    processor             *sharedProcessor
    cacheMutationDetector MutationDetector

    listerWatcher ListerWatcher
    objectType    runtime.Object

    resyncCheckPeriod time.Duration

    defaultEventHandlerResyncPeriod time.Duration

    clock clock.Clock

    started, stopped bool
    startedLock      sync.Mutex

    blockDeltas sync.Mutex
}
```

```go
func NewCacheMutationDetector(name string) MutationDetector {
    if !mutationDetectionEnabled {
        return dummyMutationDetector{}
    }
    klog.Warningln("Mutation detector is enabled, this will result in memory leakage.")
    return &defaultCacheMutationDetector{name: name, period: 1 * time.Second}
}
```

```go
cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}
```

exampleInformerFactory.Start(stopCh) 就是启动所有注册在 factory 中的 informer
```go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
    f.lock.Lock()
    defer f.lock.Unlock()

    for informerType, informer := range f.informers {
        if !f.startedInformers[informerType] {
            go informer.Run(stopCh)
            // 保存已启动的 informer 列表
            f.startedInformers[informerType] = true
        }
    }
}
```

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
    fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
        KnownObjects:          s.indexer,
        EmitDeltaTypeReplaced: true,
    })

    cfg := &Config{
        Queue:            fifo,
        ListerWatcher:    s.listerWatcher,
        ObjectType:       s.objectType,
        FullResyncPeriod: s.resyncCheckPeriod,
        RetryOnError:     false,
        ShouldResync:     s.processor.shouldResync,

        Process:           s.HandleDeltas,
        WatchErrorHandler: s.watchErrorHandler,
    }

    func() {
        s.startedLock.Lock()
        defer s.startedLock.Unlock()

        s.controller = New(cfg)
        s.controller.(*controller).clock = s.clock
        s.started = true
    }()

    // Separate stop channel because Processor should be stopped strictly after controller
    processorStopCh := make(chan struct{})
    var wg wait.Group
    defer wg.Wait()              // Wait for Processor to stop
    defer close(processorStopCh) // Tell Processor to stop
    wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
    wg.StartWithChannel(processorStopCh, s.processor.run)

    defer func() {
        s.startedLock.Lock()
        defer s.startedLock.Unlock()
        s.stopped = true // Don't want any new listeners
    }()
    s.controller.Run(stopCh)
}
```


启动 sharec:watch


informer 通知
controller 队列

Reference:

