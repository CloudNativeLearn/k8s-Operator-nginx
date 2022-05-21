#  Kubernetes Operator nginx 进阶

参考文章  https://www.qikqiak.com/post/k8s-operator-101/

##个人介绍 

方玉龙 湖北大学 大三学弟 兴趣k8s 欢迎交流

##联系方式 

qq 3095329264
邮箱 3095329264@qq.com 

## 目标

我们平时在部署一个简单的 Webserver 到 Kubernetes 集群中的时候，都需要先编写一个 Deployment 的控制器，然后创建一个 Service 对象，通过 Pod 的 label 标签进行关联，最后通过 Ingress 或者 type=NodePort 类型的 Service 来暴露服务，每次都需要这样操作，是不是略显麻烦，我们就可以创建一个自定义的资源对象，通过我们的 CRD 来描述我们要部署的应用信息，比如镜像、服务端口、环境变量等等，然后创建我们的自定义类型的资源对象的时候，通过控制器去创建对应的 Deployment 和 Service，是不是就方便很多了，相当于我们用一个资源清单去描述了 Deployment 和 Service 要做的两件事情。

这里我们将创建一个名为 AppService 的 CRD 资源对象，然后定义如下的资源清单进行应用部署：

```yaml
apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```

通过这里的自定义的 AppService 资源对象去创建副本数为2的 Pod，然后通过 nodePort=30002 的端口去暴露服务，接下来我们就来一步一步的实现我们这里的这个简单的 Operator 应用。



## 创建项目

```shell
# 创建项目目录
$ mkdir -p operator-learning  
# 设置项目目录为 GOPATH 路径
$ cd operator-learning && export GOPATH=$PWD  
$ mkdir -p $GOPATH/src/github.com/cnych 
$ cd $GOPATH/src/github.com/cnych
# 使用 sdk 创建一个名为 opdemo 的 operator 项目
$ operator-sdk new opdemo
......
# 该过程需要科学上网，需要花费很长时间，请耐心等待
......
$ cd opdemo && tree -L 2
.
├── Gopkg.lock
├── Gopkg.toml
├── build
│   ├── Dockerfile
│   ├── _output
│   └── bin
├── cmd
│   └── manager
├── deploy
│   ├── crds
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── pkg
│   ├── apis
│   └── controller
├── vendor
│   ├── cloud.google.com
│   ├── contrib.go.opencensus.io
│   ├── github.com
│   ├── go.opencensus.io
│   ├── go.uber.org
│   ├── golang.org
│   ├── google.golang.org
│   ├── gopkg.in
│   ├── k8s.io
│   └── sigs.k8s.io
└── version
    └── version.go

23 directories, 8 files
```

到这里一个全新的 Operator 项目就新建完成了。

## 项目结构

使用`operator-sdk new`命令创建新的 Operator 项目后，项目目录就包含了很多生成的文件夹和文件。

- **Gopkg.toml Gopkg.lock** — Go Dep 清单，用来描述当前 Operator 的依赖包。
- **cmd** - 包含 main.go 文件，使用 operator-sdk API 初始化和启动当前 Operator 的入口。
- **deploy** - 包含一组用于在 Kubernetes 集群上进行部署的通用的 Kubernetes 资源清单文件。
- **pkg/apis** - 包含定义的 API 和自定义资源（CRD）的目录树，这些文件允许 sdk 为 CRD 生成代码并注册对应的类型，以便正确解码自定义资源对象。
- **pkg/controller** - 用于编写所有的操作业务逻辑的地方
- **vendor** - golang vendor 文件夹，其中包含满足当前项目的所有外部依赖包，通过 go dep 管理该目录。

我们主要需要编写的是**pkg**目录下面的 api 定义以及对应的 controller 实现。

## 添加Api

```shell
operator-sdk add api --api-version=app.example.com/v1 --kind=AppService
```

输出

```powershell
INFO[0000] Generating api version app.example.com/v1 for kind AppService. 
INFO[0000] Created pkg/apis/app/group.go                
INFO[0004] Created pkg/apis/app/v1/appservice_types.go  
INFO[0005] Created pkg/apis/addtoscheme_app_v1.go       
INFO[0005] Created pkg/apis/app/v1/register.go          
INFO[0005] Created pkg/apis/app/v1/doc.go               
INFO[0005] Created deploy/crds/app.example.com_v1_appservice_cr.yaml 
INFO[0005] Running deepcopy code-generation for Custom Resource group versions: [app:[v1], ] 
INFO[0026] Code-generation complete.                    
INFO[0026] Running CRD generator.                       
INFO[0026] CRD generation complete.                     
INFO[0026] API generation complete.                     
INFO[0026] API generation complete. 
```

## 添加控制器

```shell
 operator-sdk add controller --api-version=app.example.com/v1 --kind=AppService
```

输出

```powershell
INFO[0000] Generating controller version app.example.com/v1 for kind AppService. 
INFO[0000] Created pkg/controller/appservice/appservice_controller.go 
INFO[0000] Created pkg/controller/add_appservice.go     
INFO[0000] Controller generation complete.  
```

## 自定义 API

代码中会涉及到一些包名的导入，由于包名较多，所以我们会使用一些别名进行区分，主要的包含下面几个：

```go
import (
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    appv1 "github.com/cnych/opdemo/pkg/apis/app/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```

修改 pkg/apis/v1/appservice_types.go 文件

```go
type AppServiceSpec struct {
	Size  	  *int32                      `json:"size"`
	Image     string                      `json:"image"`
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
	Envs      []corev1.EnvVar             `json:"envs,omitempty"`
	Ports     []corev1.ServicePort        `json:"ports,omitempty"`
}
```

```go
type AppServiceStatus struct {
	appsv1.DeploymentStatus `json:",inline"`
}
```

这里的 resources、envs、ports 的定义都是直接引用的`"k8s.io/api/core/v1"`中定义的结构体，而且需要注意的是我们这里使用的是`ServicePort`，而不是像传统的 Pod 中定义的 ContanerPort，这是因为我们的资源清单中不仅要描述容器的 Port，还要描述 Service 的 Port。

然后一个比较重要的结构体`AppServiceStatus`用来描述资源的状态，当然我们可以根据需要去自定义状态的描述，我这里就偷懒直接使用 Deployment 的状态了

定义完成后，在项目根目录下面执行如下命令：

```shell
$ operator-sdk generate k8s
$ operator-sdk generate crds
```

改命令是用来根据我们自定义的 API 描述来自动生成一些代码，目录`pkg/apis/app/v1/`下面以`zz_generated`开头的文件就是自动生成的代码，里面的内容并不需要我们去手动编写。

这样我们就算完成了对自定义资源对象的 API 的声明。

## 实现controller逻辑



上面 API 描述声明完成了，接下来就需要我们来进行具体的业务逻辑实现了，编写具体的 controller 实现，打开源文件`pkg/controller/appservice/appservice_controller.go`，需要我们去更改的地方也不是很多，核心的就是`Reconcile`方法，该方法就是去不断的 watch 资源的状态，然后根据状态的不同去实现各种操作逻辑，核心代码如下：

```go
func (r *ReconcileAppService) Reconcile(request reconcile.Request) (reconcile.Result, error) {
	reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
	reqLogger.Info("Reconciling AppService")

	// Fetch the AppService instance
	instance := &appv1.AppService{}
	err := r.client.Get(context.TODO(), request.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		return reconcile.Result{}, err
	}

	if instance.DeletionTimestamp != nil {
		return reconcile.Result{}, err
	}

	// 如果不存在，则创建关联资源
	// 如果存在，判断是否需要更新
	//   如果需要更新，则直接更新
	//   如果不需要更新，则正常返回

	deploy := &appsv1.Deployment{}
	if err := r.client.Get(context.TODO(), request.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
		// 创建关联资源
		// 1. 创建 Deploy
		deploy := resources.NewDeploy(instance)
		if err := r.client.Create(context.TODO(), deploy); err != nil {
			return reconcile.Result{}, err
		}
		// 2. 创建 Service
		service := resources.NewService(instance)
		if err := r.client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 3. 关联 Annotations
		data, _ := json.Marshal(instance.Spec)
		if instance.Annotations != nil {
			instance.Annotations["spec"] = string(data)
		} else {
			instance.Annotations = map[string]string{"spec": string(data)}
		}

		if err := r.client.Update(context.TODO(), instance); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}

	oldspec := appv1.AppServiceSpec{}
	if err := json.Unmarshal([]byte(instance.Annotations["spec"]), oldspec); err != nil {
		return reconcile.Result{}, err
	}

	if !reflect.DeepEqual(instance.Spec, oldspec) {
		// 更新关联资源
		newDeploy := resources.NewDeploy(instance)
		oldDeploy := &appsv1.Deployment{}
		if err := r.client.Get(context.TODO(), request.NamespacedName, oldDeploy); err != nil {
			return reconcile.Result{}, err
		}
		oldDeploy.Spec = newDeploy.Spec
		if err := r.client.Update(context.TODO(), oldDeploy); err != nil {
			return reconcile.Result{}, err
		}

		newService := resources.NewService(instance)
		oldService := &corev1.Service{}
		if err := r.client.Get(context.TODO(), request.NamespacedName, oldService); err != nil {
			return reconcile.Result{}, err
		}
		oldService.Spec = newService.Spec
		if err := r.client.Update(context.TODO(), oldService); err != nil {
			return reconcile.Result{}, err
		}

		return reconcile.Result{}, nil
	}

	return reconcile.Result{}, nil

}
```

上面就是业务逻辑实现的核心代码，逻辑很简单，就是去判断资源是否存在，不存在，则直接创建新的资源，创建新的资源除了需要创建 Deployment 资源外，还需要创建 Service 资源对象，因为这就是我们的需求，当然你还可以自己去扩展，比如在创建一个 Ingress 对象。更新也是一样的，去对比新旧对象的声明是否一致，不一致则需要更新，同样的，两种资源都需要更新的。

另外两个核心的方法就是上面的`resources.NewDeploy(instance)`和`resources.NewService(instance)`方法，这两个方法实现逻辑也很简单，就是根据 CRD 中的声明去填充 Deployment 和 Service 资源对象的 Spec 对象即可。

NewDeploy 方法实现如下：

```go
func NewDeploy(app *appv1.AppService) *appsv1.Deployment {
	labels := map[string]string{"app": app.Name}
	selector := &metav1.LabelSelector{MatchLabels: labels}
	return &appsv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "apps/v1",
			Kind:       "Deployment",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,

			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group: v1.SchemeGroupVersion.Group,
					Version: v1.SchemeGroupVersion.Version,
					Kind: "AppService",
				}),
			},
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: app.Spec.Size,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					Containers: newContainers(app),
				},
			},
			Selector: selector,
		},
	}
}

func newContainers(app *v1.AppService) []corev1.Container {
	containerPorts := []corev1.ContainerPort{}
	for _, svcPort := range app.Spec.Ports {
		cport := corev1.ContainerPort{}
		cport.ContainerPort = svcPort.TargetPort.IntVal
		containerPorts = append(containerPorts, cport)
	}
	return []corev1.Container{
		{
			Name: app.Name,
			Image: app.Spec.Image,
			Resources: app.Spec.Resources,
			Ports: containerPorts,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Env: app.Spec.Envs,
		},
	}
}
```

newService 对应的方法实现如下：

```go
func NewService(app *v1.AppService) *corev1.Service {
	return &corev1.Service {
		TypeMeta: metav1.TypeMeta {
			Kind: "Service",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name: app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group: v1.SchemeGroupVersion.Group,
					Version: v1.SchemeGroupVersion.Version,
					Kind: "AppService",
				}),
			},
		},
		Spec: corev1.ServiceSpec{
			Type: corev1.ServiceTypeNodePort,
			Ports: app.Spec.Ports,
			Selector: map[string]string{
				"app": app.Name,
			},
		},
	}
}
```

这样我们就实现了 AppService 这种资源对象的业务逻辑。

## 开始部署

修改  deploy/crds/app_v1_appservice_cr.yaml文件

```go
apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```

 operator-sdk build mock.com:5000/imoocpod-nginx

docker push mock.com:5000/imoocpod-nginx

kubectl apply -f deploy/service_account.yaml 

 kubectl apply -f deploy/role.yaml 

 kubectl apply -f deploy/role_binding.yaml 

 kubectl apply -f deploy/crds/k8s.imooc.llcom_imoocpods_crd.yaml

替换operator.yaml 中的image

 kubectl apply -f deploy/operator.yaml

kubectl apply -f deploy/crds/k8s.imooc.com_v1alpha1_imoocpod_cr.yaml

```powershell
fangyulong@BDSZYF000146577 crds % kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
nginx-app-6dd547f6f-6fgpm   1/1     Running   0          13m
nginx-app-6dd547f6f-ks8kp   1/1     Running   0          13m
opdemo-769464d9dd-7rh2p     1/1     Running   0          14m
fangyulong@BDSZYF000146577 crds % 

```

搭建完成

