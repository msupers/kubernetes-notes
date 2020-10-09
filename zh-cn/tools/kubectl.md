# 登录容器

?> 目前容器不启动ssh服务。登录容器请使用kubectl登录的方式，具体方法如下

## 一、登录到跳板机

```
# 用户名 k8s
# 密码 123456.com
ssh k8s@172.24.29.43
```

## 二、找到要登录的容器

?>格式：kubectl get pods -n `NAMESPACE` |grep `子项目英文名`(大写字母换成小写)

```shell
# 实例：

kubectl get pods -n tess-test-p20-069-tess-dev |grep cis-backend-demo

cis-backend-demo-69b76ddf99-tgx68     1/1     Running   0          41h
```

## 三、登录目标容器

?>格式: kubectl exec -it `容器名称` bash -n `NAMESAPCE`

```shell
#实例：

kubectl exec -it cis-backend-demo-69b76ddf99-tgx68 bash -n tess-test-p20-069-tess-dev

root@cis-backend-demo-69b76ddf99-tgx68:/usr/local/tomcat/webapps#
```

