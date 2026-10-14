# 官方链接  
1、GitHub：https://github.com/argoproj/argo-cd/  
2、官  网：https://argo-cd.readthedocs.io/

# 部署
## 1. 创建namespace  
```
kubectl create ns argocd
```
## 2. 应用清单  
```
kubectl apply -f https://gitee.com/zqli6/cicd/raw/main/argocd/manifest/argocd-v3.5.5-lzq.yaml --server-side --force-conflicts -n argocd
```

## 3. pull secret patch  
参考：https://gitee.com/zqli6/HA-cluster/blob/master/SWR-AUTH.md#%E5%A4%9A%E4%B8%AApod%E5%90%8C%E6%97%B6
```
# 1. 创建 Secret（存储 SWR 登录凭据）需登录swr 更新
kubectl create secret docker-registry swr-auth -n argocd \
  --docker-server=swr.cn-southwest-2.myhuaweicloud.com \
  --docker-username='cn-southwest-2@HST3WBRU8484RY3AGAFX' \
  --docker-password='8eb15bd0873dc162d8be3057ba45653764d344d3926484c1bbf1e1836b2a01cd'

# 2. 绑定到 ServiceAccount
kubectl get pod -n argocd -o jsonpath='{range .items[*]}{.spec.serviceAccountName}{"\n"}{end}' \
  | sort -u \
  | while read sa; do
      [ -z "$sa" ] && continue
      echo ">>> patching SA: $sa"
      kubectl patch serviceaccount "$sa" -n argocd \
        -p '{"imagePullSecrets": [{"name": "swr-auth"}]}'
    done
# 3. 删除pod重新拉取镜像
kubectl delete pod -n argocd --all
```
## 4. 登录  
访问：https://ip:30443  
账号：admin  
密码：
```
kubectl get secrets -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
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