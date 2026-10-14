# 官方链接  
1、GitHub：https://github.com/argoproj/argo-cd/  
2、官  网：https://argo-cd.readthedocs.io/

# 部署
1. 创建namespace  
```
kubectl create ns argocd
```
2. 应用清单  
```
kubectl apply -f https://gitee.com/zqli6/cicd/raw/main/argocd/manifest/argocd-v3.5.5-lzq.yaml --server-side --force-conflicts -n argocd
```

# 优化说明
## 1 镜像  
```
root@master1:/data/argocd# grep image: argocd-v3.5.5-lzq.yaml | sort -u
        image: ghcr.io/dexidp/dex:v2.45.1
        image: public.ecr.aws/docker/library/redis:8.2.3-alpine
        image: quay.io/argoproj/argocd:v3.5.5
```
```
swr.cn-southwest-2.myhuaweicloud.com/zqli/ghcr.io/dexidp/dex:v2.45.1
swr.cn-southwest-2.myhuaweicloud.com/zqli/quay.io/argoproj/argocd:v3.5.5
# 使用redis:7.2.5代替public.ecr.aws/docker/library/redis:8.2.3-alpine
swr.cn-southwest-2.myhuaweicloud.com/zqli/redis:7.2.5
```
## 2 配置  
### 1. 修改argocd-server的service为NodePort:30080及30443
```
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
  name: argocd-server
spec:
  # 默认没有type，默认为ClusterIP
  type: NodePort       # lzq清单优化
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
    nodePort: 30080    # lzq优化
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
    nodePort: 30443    # lzq优化
  selector:
    app.kubernetes.io/name: argocd-server
---
```