---
layout: post
title: "[庖丁解牛]一文读透kubernetes controller编写要点"
date:   2025-02-08
tags: [kubernetes, controller, 庖丁解牛]
comments: false
author: 小辣条
toc : true
---

最近在运维kubesphere平台的过程中，发现其组件中的devops-controller在处理大量事件时，会出现处理延迟，导致一些事件无法及时处理。经过查阅资料，发现可以通过调整controller的并发处理能力来提高事件处理效率。借此机会，总结了下kubernetes controller编写要点。

<!-- more -->
# 一、配置controller定时调谐和频率

尽管 Kubernetes 设计为级别触发，但协调器可在级别和边缘触发下工作，因为观察者希望尽快收到对象变化事件的通知。

默认情况下，当协调器所监视的任何类型的对象发生变化时，都会触发协调器。此外，还可以将其配置为定期在所有对象上运行。这可以通过两种方式完成：

在创建新管理器时，在控制器运行时选项中设置SyncPeriod（默认为 10 小时，运行抖动为 10%）
处理完对象后，从协调器返回Result.RequeueAfter 。
定期协调器将按照指定的时间间隔对对象调用 Reconcile 方法，而不管对象是否发生任何变化。这应该针对两种用例完成：

确保不会因错过监视事件而出现协调器错误。
轮询无法监视的对象或基于外部事件的操作。
创建管理器时可以设置 SyncPeriod 选项，如下所示。

```
func main() { 
    s := time.Minute * 10 
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{ 
        // 其他选项 ... 
        SyncPeriod: &s, // 默认为 10 小时，抖动为 10% 
    })
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil { 
        setupLog.Error(err, "运行管理器时出现问题") 
        os.Exit(1) 
    } 
}
func (r *MyReconciler) SetupWithManager(mgr ctrl.Manager) error { 
    return ctrl.NewControllerManagedBy(mgr). 
        For(&mypackage.MyObject{}).   
        Complete(r) 
}
```
可以在 Reconciler 返回对象中设置 RequeueAfter 持续时间，并且即使对象没有变化，Reconciler 也会在该持续时间后再次运行。
```
import (
    "sigs.k8s.io/controller-runtime/pkg/client"
    ctrl "sigs.k8s.io/controller-runtime"
)
func (r *ApiVersionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {    
    var obj mypackage.MyObject
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        log.Errorf("error in getting my object : %+v", err)
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // reconciler logic goes here...
    return ctrl.Result{RequeueAfter: time.Minute * 10}, nil
}
```
因此，SyncPeriod 和 RequeueAfter 都会导致周期性协调。但建议保留 SyncPeriod 的默认值，并使用 RequeueAfter 来实现周期性协调。

# 二、如何从controller优雅返回
本部分中，让我们研究从协调器返回的各种方式
```
func (r *MyReconciler) Reconcile(ctx context.Context, req    ctrl.Request) (ctrl.Result, error) {    
    // log error and requeue with rate limiting
    return ctrl.Result{}, err    
    // requeue with rate limiting without logging any error
    return ctrl.Result{Requeue: true}, nil     
    // requeue after time.Duration (Result.Requeue is ignored)
    return ctrl.Result{RequeueAfter: time.Duration}, nil    
    // do no requeue the reconciler 
    return ctrl.Result{}, nil
}
```
如果协调器因删除事件而触发，则r.Get调用将出错，如果协调器在这种情况下返回错误，则会导致无限循环。为了避免这种情况，检查r.Get错误很重要，如果它属于未找到类型，则必须从协调器返回 nil 以停止协调器。可以使用控制器运行时函数 client.IgnoreNotFound(err) 轻松处理此问题。
```
import (
    "sigs.k8s.io/controller-runtime/pkg/client"
    ctrl "sigs.k8s.io/controller-runtime"
)
func (r *ApiVersionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {    
    var obj mypackage.MyObject
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        log.Errorf("error in getting my object : %+v", err)
        // returning error here would lead to infinite loop
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }    
    return ctrl.Result{}, nil
}
```
为了避免在出现 NotFound 错误时出现错误日志，请进行条件检查并相应地记录。
```
import (
    "sigs.k8s.io/controller-runtime/pkg/client"
    ctrl "sigs.k8s.io/controller-runtime"
    "k8s.io/apimachinery/pkg/api/errors"
)
func (r *ApiVersionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {    
    var obj mypackage.MyObject
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        if errors.IsNotFound(err) {
            log.Infof("my object not found, may have been deleted")
            return ctrl.Result{}, nil
        }
        log.Errorf("error in getting my object : %+v", err)
        return ctrl.Result{}, err
    }
    // reconciler logic goes here ...
    return ctrl.Result{}, nil
}
```

# 三、如何处理资源被删除场景
在本部分中，让我们看看如何处理 Kubernetes 对象的删除。

我们知道 Kubernetes 控制器模式基于状态而非事件。因此，协调器循环无法告诉我们当前调用的是哪个 CRUD 操作。

可以使用当前状态来管理 CRU，但是我们希望知道对象是否被删除，以便执行相关的清理。

有一种方法可以确定事件是否由于对象删除而发生。删除对象后，将在对象的ObjectMeta中设置DeletionTimestamp 。

但是在删除一个对象并由于该删除而调用协调器之后，调用r.Get(ctx, req.NamespacedName, &obj)将失败并出现未找到错误，因为该对象在内部被立即删除。

为了解决这个问题，我们必须通知 Kubernetes 不要立即删除该对象，让我们最后一次完成 Reconciler。这可以使用Finalizers来完成。

其工作原理是，在创建对象时需要设置终结器注释，并且 Kubernetes 不会删除该对象（只会将其标记为已删除），直到设置的终结器被移除。可以在最后一次协调对象时移除终结器，这可以使用ObjectMeta上的DeletionTimestamp来确定。
```
import (
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"        
    "sigs.k8s.io/controller-runtime/pkg/client"
    ctrl "sigs.k8s.io/controller-runtime"
)
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {    
    var obj mypackage.MyObject
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        log.Errorf("error in getting my object : %+v", err)
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    if err := r.handleFinalizer(ctx, obj); err != nil {
        log.Errorf("failed to update finalzier : %+v", err)
        return ctrl.Result{}, err
    }
    if !obj.ObjectMeta.DeletionTimestamp.IsZero() {
        // handle delete and return
    }
    // handle create/update    
    return ctrl.Result{}, nil
}
func (r *MyReconciler) handleFinalizer(ctx context.Context, obj package.MyObject) error {
    name := "finalizer-name"
    if obj.ObjectMeta.DeletionTimestamp.IsZero() {
        // add finalizer in case of create/update
        if !controllerutil.ContainsFinalizer(&obj, name) {
             ok := controllerutil.AddFinalizer(&obj, name)
             log.Infof("Add Finalizer %s : %t", name, ok)
             return r.Update(ctx, &obj)
        } 
    } else {
        // remove finalizer in case of deletion
        if controllerutil.ContainsFinalizer(&obj, name) {
            ok := controllerutil.RemoveFinalizer(&obj, name)
            log.Infof("Remove Finalizer %s : %t", name, ok)
            return r.Update(ctx, &obj)
        }
    }
    return nil
}
```
# 四、如何过滤监听特定事件event
在本部分中，我们将学习如何过滤事件并控制协调器循环的执行。

Predicates可用于过滤事件并控制协调器循环的执行。Predicates包中预定义了一些Predicates，可以通过实现Predicates接口来创建自定义Predicates。 

```
// Predicate 在将键入队之前过滤事件。
type Predicate接口 {
   Create ( event . CreateEvent )  bool
   Delete ( event . DeleteEvent )  bool
   Update ( event . UpdateEvent )  bool
   Generic ( event . GenericEvent )  bool 
} 

// Funcs 实现 Predicate。
type Funcs struct  {
   CreateFunc func( event . CreateEvent )  bool
   DeleteFunc func( event . DeleteEvent )  bool
   UpdateFunc func( event . UpdateEvent )  bool
   GenericFunc func( event . GenericEvent )  bool 
}
```
过滤事件的最常见用例是，仅当对象的规范发生变化时才调用协调器。这是由GenerationChangePredicate实现的。如果metadata.Generation字段没有变化，此过滤器将跳过事件，并且此字段仅在对象规范发生变化时才会更改。这通常用于当协调器更新对象的状态并且我们不希望因状态更改而调用协调器时。
```
func (r *MyReconciler) SetupWithManager(mgr ctrl.Manager) error { 
    return ctrl.NewControllerManagedBy(mgr). 
        For(&mypackage.MyObject{}). 
        WithEventFilter(predicate.GenerationChangedPredicate{}). 
        Complete(r) 
}
或
func (r * MyReconciler ) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).For 
        (&mypackage.MyObject{}).
        WithEventFilter(predicate.Funcs{ 
            UpdateFunc：func(e event.UpdateEvent) bool { 
                oldGeneration := e.MetaOld.GetGeneration() 
                newGeneration := e.MetaNew.GetGeneration()
                return oldGeneration != newGeneration 
            }, 
        })。
        Complete(r) 
}
```
WithEventFilter(predicate.GenerationChangedPredicate{})会中断使用 SyncPeriod 字段设置的定期同步。在此启动了一个线程。


# 五、controller并发调谐

在本部分中，我们将重点介绍如何并发运行协调器。

默认情况下，一次只运行一个协调器循环实例。其余实例将排队并等待当前实例完成。

如果监视对象的状态频繁变化，并且大量协调事件进入队列，则可以配置控制器以同时运行协调器循环并更快地清空处理队列。

请注意，并发处理不适用于同一对象。换句话说，协调器永远不会针对同一对象同时调用。因此，实施者不必担心因并发而可能发生的一致性问题。

这样做的潜在缺点是事件可能无序处理。例如，如果事件以 {AABC} 的顺序出现，则处理可能以 {ABCA} 的顺序进行，因为对象 A 的第二个事件被保留，直到第一个 A 的处理完成。第一个 A 完成后，第二个 A 将立即排队等待处理。

建议启用并发协调，因为单个协调器线程可能会导致事件处理延迟，因为此设计不会同时处理同一类型的事件。例如，如果事件按 {AABC} 的顺序出现，并且只有一个协调器线程，则事件将按 {ABCA} 的顺序处理，第二个 A 正在等待事件 B 和 C 完成，即使它先发生。

可以使用控制器的Options参数中的MaxConcurrentReconciles字段来设置控制器的并发值。

```
import (
    "sigs.k8s.io/controller-runtime/pkg/controller"
    "sigs.k8s.io/controller-runtime/pkg/predicate"
)
func (r *MyReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&mypackage.MyObject{}).
        WithEventFilter(predicate.GenerationChangedPredicate{}).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 10,
        }).Complete(r)
}
```

# 六、如何测试controller
## 1.使用fake client
此方法使用Kubernetes 控制器运行时库提供的fake client。

它不运行任何实际的依赖二进制文件，也不使用任何端口或网络调用来模拟正在运行的 Kubernetes 集群。

所有操作都在内存中完成，因此因真实设置而导致测试失败或不稳定的可能性较小。

它还重量轻、速度快并且占用的资源最少。

```
import (
    "sigs.k8s.io/controller-runtime/pkg/client/fake"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
)
finalizerName := "finalizer-name"
// create scheme
sch := runtime.NewScheme()
mypackage.AddToScheme(sch)
Expect(sch).ToNot(BeNil())
// create fake kubernetes client
k8sClient := fake.NewClientBuilder().WithScheme(sch).Build()
Expect(k8sClient).ToNot(BeNil())
objName := "obj-name"
objSpec := mypackage.MyObjectSpec{ // add spec fields }
obj := &mypackage.MyObject{
    ObjectMeta: metav1.ObjectMeta{Name: objName},
    Spec: objSpec,
}
// create the object
err := k8sClient.Create(ctx, obj)
Expect(err).NotTo(HaveOccurred())
// call the reconciler manually as the setup is a mock
r := &mypackage.MyObjectReconciler{Client: k8sClient}
res, err := r.Reconcile(ctx, reconcile.Request{
    NamespacedName: types.NamespacedName{
        Name: obj.Name,
    },
})
Expect(err).NotTo(HaveOccurred())
Expect(res).To(Equal(ctrl.Result{}))
// get the object to reflect updates from reconcilation
err := k8sClient.Get(ctx, types.NamespacedName{Name: objName}, obj)
Expect(err).NotTo(HaveOccurred())
// perform validations if any
Expect(obj.Status.State).To(Equal("COMPLETE"))
Expect(obj.Status.Message).To(Equal("all resources created"))
Expect(obj.Finalizers).To(ContainElement(finalizerName))

```

## 2.使用envtest
此方法使用Kubernetes controller-runtime 库提供的envtest框架。

它适用于 etcd、kubectl 和 Kube API 服务器的实际运行实例。可以使框架指向现有的 Kubernetes 集群，或者框架可以运行这些二进制文件进行测试。可以使用环境变量KUBEBUILDER_ASSETS 设置这些二进制文件的路径。

```
import (
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    ctrl "sigs.k8s.io/controller-runtime"
)
finalizerName := "finalizer-name"
// location of etcd, kubectl and kube-apiserver binaries
os.Setenv("KUBEBUILDER_ASSETS", "../testbin/bin/"))
// create scheme
sch := runtime.NewScheme()
err := mypackage.AddToScheme(sch)
Expect(err).ToNot(HaveOccurred())
// CRD paths which need to be installed in the test environment
crdPaths := []string{filepath.Join("..", "testbin", "crds")}
// setup test environment
testEnv := &envtest.Environment{
    Scheme:                sch,
    CRDDirectoryPaths:     crdPaths,
    ErrorIfCRDPathMissing: true,
}
// start test environment
cfg, err := testEnv.Start()
Expect(err).NotTo(HaveOccurred())
Expect(cfg).NotTo(BeNil())
// create kubernetes client attached to the test environment
k8sClient, err := client.New(cfg, client.Options{})
Expect(err).NotTo(HaveOccurred())
Expect(k8sClient).NotTo(BeNil())
// create manager
mgr, err := ctrl.NewManager(cfg, ctrl.Options{Scheme: scheme})
Expect(err).NotTo(HaveOccurred())
Expect(mgr).NotTo(BeNil())
// initialize reconciler with manager
if err = (&mypackage.MyReconciler{
    Client: mgr.GetClient(),
    Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
    setupLog.Error(err, "creation error", "controller", "MyObject")
    os.Exit(1)
}
objName := "obj-name"
obj := &mypackage.MyObject{
    ObjectMeta: metav1.ObjectMeta{Name: objName},
    Spec: mypackage.MyObjectSpec{ // add spec fields },
}
// create the object
err := k8sClient.Create(ctx, obj)
Expect(err).NotTo(HaveOccurred())
// reconciler is called automatically as this is not a mock and 
// works with real binaries
// get the object to reflect updates from reconcilation
err := k8sClient.Get(ctx, types.NamespacedName{Name: objName}, obj)
Expect(err).NotTo(HaveOccurred())
// perform validations if any
Expect(obj.Status.State).To(Equal("COMPLETE"))
Expect(obj.Status.Message).To(Equal("all resources created"))
Expect(obj.Finalizers).To(ContainElement(finalizerName))
// close environment and perform cleanup
Expect(testEnv.Stop()).NotTo(HaveOccurred())
Expect(os.Unsetenv("KUBEBUILDER_ASSETS")).To(Succeed())
```

# 参考
[Kubernetes Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)

[Kubebuilder Book](https://book.kubebuilder.io/)

[Operator Framework](https://sdk.operatorframework.io/docs/)

[learning-concurrent-reconciling](https://openkruise.io/blog/learning-concurrent-reconciling/)

[writing-a-custom-controller-in-kubernetes](https://blog.stackademic.com/writing-a-custom-controller-in-kubernetes-970e3196739f)

[controller-runtime-client-go-rate-limiting](https://danielmangum.com/posts/controller-runtime-client-go-rate-limiting/)
