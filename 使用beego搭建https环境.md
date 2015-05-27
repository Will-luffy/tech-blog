#使用beego搭建https环境
##密钥生成
首先需要生成测试用的公钥和私钥
openssl genrsa -out key.pem 2048
openssl req -new -x509 -key key.pem -out cert.pem -days 3650
## 修改beego配置文件
修改app.conf
```go
EnableHttpTLS = true
HttpsPort = 10443 // 默认即10443
HttpCertFile = ${path}/cert.pem // certfile路径
HttpKeyFile = ${path}/key.pem // keyfile路径
```
## 运行beego
bee run
