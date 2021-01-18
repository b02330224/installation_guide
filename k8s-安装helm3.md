# k8s-安装helm3

# helm安装:

```shell
[root@web-test-01 ~]
wget https://get.helm.sh/helm-v3.1.2-linux-amd64.tar.gz
tar zxvf helm-v3.1.2-linux-amd64.tar.gz
cd linux-amd64
cp helm /usr/bin/helm
helm version
version.BuildInfo{Version:"v3.1.2", GitCommit:"d878d4d45863e42fd5cff6743294a11d28a9abce", GitTreeState:"clean", GoVersion:"go1.13.8"}
[root@192 demo]# helm help
The Kubernetes package manager

Common actions for Helm:

- helm search:    search for charts
- helm pull:      download a chart to your local directory to view
- helm install:   upload the chart to Kubernetes
- helm list:      list releases of charts
```

 helm3不在需要tiller,可以设置环境变量KUBECONFIG来指定存有ApiServre的地址与token的配置文件地址，默认为~/.kube/config

# 查看helm配置信息:



```shell
[root@192 templates]# helm env
HELM_BIN="helm"
HELM_DEBUG="false"
HELM_KUBECONTEXT=""
HELM_NAMESPACE="default"
HELM_PLUGINS="/root/.local/share/helm/plugins"
HELM_REGISTRY_CONFIG="/root/.config/helm/registry.json"
HELM_REPOSITORY_CACHE="/root/.cache/helm/repository"
HELM_REPOSITORY_CONFIG="/root/.config/helm/repositories.yaml"
```



# 添加共有的仓库:



```shell
helm repo add stable http://mirror.azure.cn/kubernetes/charts
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 
helm repo update
```



查看配置的存储库
helm repo list

 helm search repo stable

删除存储库：
helm repo remove aliyun





```shell
#查找 chart
[root@master ~]# helm search repo weave

NAME                    CHART VERSION   APP VERSION     DESCRIPTION                                       
aliyun/weave-cloud      0.1.2                           Weave Cloud is a add-on to Kubernetes which pro...
aliyun/weave-scope      0.9.2           1.6.5           A Helm chart for the Weave Scope cluster visual...
stable/weave-cloud      0.3.7           1.4.0           Weave Cloud is a add-on to Kubernetes which pro...
stable/weave-scope      1.1.10          1.12.0          A Helm chart for the Weave Scope cluster visual...
```

```shell
#查看chart
[root@master ~]# helm show chart stable/weave-cloud
apiVersion: v1
appVersion: 1.4.0
description: |
  Weave Cloud is a add-on to Kubernetes which provides Continuous Delivery, along with hosted Prometheus Monitoring and a visual dashboard for exploring & debugging microservices
home: https://weave.works
icon: https://www.weave.works/assets/images/bltd108e8f850ae9e7c/weave-logo-512.png
maintainers:
- email: ilya@weave.works
  name: errordeveloper
- email: stefan@weave.works
  name: stefanprodan
name: weave-cloud
version: 0.3.7
```

```shell
#安装包 # helm install ui stable/weave-scope

#查看发布状态
[root@master ~]# helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
ui      default         1               2020-11-01 03:13:10.628311633 +0800 CST deployed        weave-scope-1.1.10      1.12.0    


[root@master ~]#  helm status ui 
NAME: ui
LAST DEPLOYED: Sun Nov  1 03:13:10 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
You should now be able to access the Scope frontend in your web browser, by
using kubectl port-forward:

kubectl -n default port-forward $(kubectl -n default get endpoints \
ui-weave-scope -o jsonpath='{.subsets[0].addresses[0].targetRef.name}') 8080:4040

then browsing to http://localhost:8080/.
For more details on using Weave Scope, see the Weave Scope documentation:

https://www.weave.works/docs/scope/latest/introducing/
[root@master ~]# 

#修改 service Type: NodePort 即可访问 ui

```



自定义chart

```shell
[root@master ~]# helm create mychart
Creating mychart
[root@master ~]# ls
anaconda-ks.cfg  calico-39  helm-v3.0.0-linux-amd64.tar.gz  images.list  kube-proxy.tar  linux-amd64  mychart  pause.tar  test_yaml
[root@master ~]# cd mychart/
[root@master mychart]# ls
charts  Chart.yaml  templates  values.yaml
[root@master mychart]# cd templates/

[root@master templates]# rm -rf *


[root@master templates]# kubectl create deployment web1 --image=nginx --dry-run -o yaml >deployment.yaml
[root@master templates]# kubectl expose deployment web1 --port=8081 --target-port=80 --type=NodePort --dry-run -o yaml >service.yaml

#安装
[root@master ~]# helm install web1 mychart/
NAME: web1
LAST DEPLOYED: Sun Nov  1 04:24:55 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
[root@master ~]# 



[root@master ~]# kubectl get svc -o wide
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE     SELECTOR
httpd-svc        ClusterIP   10.101.8.133     <none>        8080/TCP         147m    app=httpd
kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP          7h17m   <none>
ui-weave-scope   NodePort    10.101.124.252   <none>        80:30131/TCP     73m     app=weave-scope,component=frontend,release=ui
web1             NodePort    10.105.175.28    <none>        8081:30803/TCP   94s     app=web1


[root@master ~]# kubectl get pod web1-858cff4454-k9jnb  -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
web1-858cff4454-k9jnb   1/1     Running   0          5m13s   10.244.215.7   work1   <none>           <none>

#访问nginx
curl  work1:30803
```





```shell
#升级
[root@master ~]# helm upgrade web1 mychart/
Release "web1" has been upgraded. Happy Helming!
NAME: web1
LAST DEPLOYED: Sun Nov  1 04:32:09 2020
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

使用values.yaml文件



values.yaml文件中定义变量

replicas: 2
image: :nginx
tag: 1.16
label: nginx
port: 80



```yaml
[root@master templates]# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: {{.Values.label}} 
  name: {{.Release.Name}}-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{.Values.label}}
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: {{.Values.label}}
    spec:
      containers:
      - image: {{.Values.image}}
        name: nginx
        resources: {}
status: {}

[root@master templates]# cat service.yaml 
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: {{.Values.label}} 
  name: {{.Release.Name}}-svc
spec:
  ports:
  - port: {{.Values.port}} 
    protocol: TCP
    targetPort: 80
  selector:
    app: {{.Values.label}} 
  type: NodePort
status:
  loadBalancer: {}
  
  
  [root@master ~]# helm install --dry-run web2 mychart/
NAME: web2
LAST DEPLOYED: Sun Nov  1 05:05:01 2020
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: mychart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx 
  name: web2-svc
spec:
  ports:
  - port: 80 
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx 
  type: NodePort
status:
  loadBalancer: {}
---
# Source: mychart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx 
  name: web2-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}



```

```shell
[root@master ~]# helm install web2 mychart/
NAME: web2
LAST DEPLOYED: Sun Nov  1 05:05:26 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
[root@master ~]# helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
ui      default         1               2020-11-01 03:13:10.628311633 +0800 CST deployed        weave-scope-1.1.10      1.12.0     
web1    default         2               2020-11-01 04:32:09.125507454 +0800 CST deployed        mychart-0.1.0           1.16.0     
web2    default         1               2020-11-01 05:05:26.782334719 +0800 CST deployed        mychart-0.1.0           1.16.0     
[root@master ~]# kubectl get pod
NAME                                            READY   STATUS    RESTARTS   AGE
busybox-5cf647b64-tjh5m                         1/1     Running   0          3h5m
httpd-57c7d78848-7fwmg                          1/1     Running   0          3h16m
httpd-57c7d78848-zzpdf                          1/1     Running   0          3h16m
weave-scope-agent-ui-5rs4r                      1/1     Running   0          112m
weave-scope-agent-ui-dhxvv                      1/1     Running   0          112m
weave-scope-agent-ui-l4xf9                      1/1     Running   0          112m
weave-scope-cluster-agent-ui-6cd95f9f76-8rdbk   1/1     Running   0          112m
weave-scope-frontend-ui-66dff7b5c6-qjg67        1/1     Running   0          112m
web1-858cff4454-k9jnb                           1/1     Running   0          40m
web2-deploy-86c57db685-7xntx                    1/1     Running   0          19s
```

