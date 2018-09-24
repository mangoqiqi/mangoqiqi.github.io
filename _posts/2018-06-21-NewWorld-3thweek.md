---
layout:     post
title:      实习工作（三）
subtitle:   记录每周学习的内容
date:       2018-09-23
author:     Hxd
header-img: img/city.jpg
catalog: true
tags: 实习
---

## 星期一

了解了hfs服务器的原理。

## 星期二

自建yum源，使用golang搭建了http文件服务器，主要功能和nginx差不多，由于ngix安装起来麻烦并且容易出错，所以用go程序代替，主要代码如下：
```
func StartSever(path, port string) error  {
	http.Handle("/", http.FileServer(http.Dir(path)))
	log.Infof("server starting on %s,expose path at %s",port,path)
	err := http.ListenAndServe(fmt.Sprint(":",port), nil)
	if err != nil {
		log.Error(err)
	}
	return err
}
```
起一个server监听某端口就可以。

## 星期三

弄了自建registry,在k8s集群的一个机器中创建本地registry服务，并且自带签名。

```

1.docker pull registry
2.docker save registry.tar
3.docker load < registry.tar
4.docker tag registry.saas.hand-china.com/google_containers/defaultbackend:1.4 172.17.31.119:5000/test

5.docker pull 172.17.31.119:5000/test
6.vim  /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd  --insecure-registry 192.168.0.153:5000

7.systemctl restart docker
8.docker push 172.17.31.119:5000/test

9.echo '{ "insecure-registries":["172.17.31.119:5000"] }' > /etc/docker/daemon.json
10.systemctl restart docker
11.docker pull 172.17.31.119:5000/test
```


## 星期四

弄根证书和自签名。
```

生成根证书密钥：
openssl genrsa -des3 -out rootCA.key 2048
生成根证书
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.pem
信任
cat rootCA.pem >> /etc/pki/tls/certs/ca-bundle.crt

//cp rootCA.pem /etc/docker/certs.d/domain\:5000/ca.crt  docker的push pull认证

创建一个v3.ext文件，以创建一个X509 v3证书。
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName=@alt_names
[alt_names]
DNS.1 = doi.com


创建证书密钥:

openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key
创建子证书：
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 500 -sha256 -extfile v3.ext

//cp server.crt /etc/docker/certs.d/domain\:5000/ca.crt  docker的push pull认证

跑registry：
docker run -d -p 5000:5000 \
-v ~/rootCA:/certs \
-v /etc/localtime:/etc/localtime:ro \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/server.key \
--restart=always --name registry registry:2

修改hosts：
vi /etc/hosts



添加认证之后：
docker run -d -p 5000:5000 \
-v ~/rootCA:/certs \
-v /opt/docker/registry/auth:/auth \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-v /etc/localtime:/etc/localtime:ro \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/server.key \
--restart=always --name registry registry:2

```


## 星期五

今天完成了搭建了单节点k8s集群离线部署.


## 总结
这周做的工作不算难，主要的难点就是证书的自签发和和优化离线安装的思路。原来的办法是，每一台机器山都要执行镜像的load的操作，这里做的改进就是，在一台机器里搭建本地registry仓库，其他镜像只要从这台机器拉取镜像就好，但是，docker拉取镜像要使用https保证连接安全，这就需要自签名。这里采用直接签发根证书的方法，其他机器信任此根证书之后用此根证书签发的子证书都会被信任。