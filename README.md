# 获取Jenkins插件  
```
docker pull swr.cn-southwest-2.myhuaweicloud.com/zqli/jenkins-plugins:26.3.17.15
docker pull swr.cn-southwest-2.myhuaweicloud.com/zqli/jenkins-plugins:docker_docker_pipeline_k8s
```
```
docker create --name plugins swr.cn-southwest-2.myhuaweicloud.com/zqli/jenkins-plugins:26.3.17.15 echo
docker create --name plugins-k9s swr.cn-southwest-2.myhuaweicloud.com/zqli/jenkins-plugins:docker_docker_pipeline_k8s
dcoker cp plugins:/data/jenkins-plugins.tar.gz ./jenkins-plugins.tar.gz
dcoker cp plugins-k8s:/data/jenkins-plugins.tar.gz ./jenkins-plugins-k8s.tar.gz
tar xf jenkins-plugins.tar.gz -C /var/lib/jenkins/plugins
tar xf jenkins-plugins-k8s.tar.gz -C /var/lib/jenkins/plugins
```