### grpc(golang) 整理

#### 环境配置

##### 1.安装protoc

```
下载 & 安装 

https://github.com/protocolbuffers/protobuf/releases/download/v3.12.0/protoc-3.12.0-osx-x86_64.zip

解压

mv ./bin/protoc /usr/local/bin
mv include/*  /usr/local/include

测试: protoc

```



##### 2.安装golang语言模块

```
git clone https://github.com/golang/protobuf/
cd protoc-gen-go
go build

mv protoc-gen-go /usr/local/bin
```



##### 3.安装grpc代码生成插件

``` 
参考: https://github.com/grpc/grpc-go/blob/master/README.md

git clone https://github.com/grpc/grpc-go.git $GOPATH/src/google.golang.org/grpc

go mod edit -replace=google.golang.org/grpc=github.com/grpc/grpc-go@latest
go mod tidy
go mod vendor
go build -mod=vendor

```



















