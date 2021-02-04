# 使用helm安装sonarqube



helm chart 官网 sonarqube chart 地址

https://github.com/helm/charts/tree/master/stable/sonarqube

该chart已经废弃，新的地址如下

https://github.com/Oteemo/charts/tree/master/charts/sonarqube

添加 chart 仓库
```shell
helm repo add oteemocharts https://oteemo.github.io/charts
```
搜索chart
```shell
helm search repo sonar
```
下载chart
```shell
helm fetch oteemocharts/sonarqube
```

根据chart生成yaml部署文件（安装中文语言包插件，中文语言包：https://github.com/xuhuisheng/sonar-l10n-zh）

```shell
helm template sonar-community oteemocharts/sonarqube --set service.type=NodePort --set plugins.install[0]="https://github.com/xuhuisheng/sonar-l10n-zh/releases/download/sonar-l10n-zh-plugin-8.5/sonar-l10n-zh-plugin-8.5.jar" > sonar-community.yaml
```

注意：plugins.install 的参数是数组

部署
```shell
kubectl create ns sonar-community
kubectl create -f sonar-community1.yaml -n sonar-community
```

