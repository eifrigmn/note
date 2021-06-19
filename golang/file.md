关于文件传输

上传文件时，使用form-data提交文件，可以设定参数名为`files`上传多个文件

## 文件上传

`*http.Request`获取文件内容并写入磁盘

+ MultipartReader()

~~~go
r, _ := http.Request.MultipartReader()
~~~

然后循环读取

~~~go
r.NextPart()
~~~

直到`err == EOF`

+ c.MultipartForm()

gin框架提供这样一种方法

~~~go
form, err := c.MultipartForm()
~~~

可以从中获取到

~~~go
files := form.File["files"]
~~~

然后解析每一个文件的文件名，文件内容等，写入磁盘

## 增加业务功能

在完成文件上传的基础功能后，针对大文件，重复上传文件、分布式服务等情况，对服务进行优化

### 秒传

服务端记录文件的哈希值，客户端上传文件前先发送待上传文件的哈希值，若文件哈希值已存在，则视为相同文件，待上传的文件已上传成功，即为秒传

这里，服务端可以借用levelDB、Redis等数据库来记录文件与文件哈希值的对应关系，方便即时查找指定哈希值

### 断点续传

客户端在上传文件的同时，提供参数指定读取文件的偏移量，服务端可以跳过已上传过的文件内容，接着"上一次"数据传输的进度继续读取文件。

可以参考一个断点续传的协议[tus](https://tus.io/)

#### 引申

是否可以在客户端上传文件的同时，提供文件的哈希值，服务端按照文件哈希值记录文件内容以及文件上传的进度，这样可以在服务端控制断点续传

### 分块上传

客户端对大文件进行分割，再请求服务端上传文件，服务端接收完文件块后，可以分块存储，或合并后存储，如此可以提高文件上传的效率。

### 参考项目

[fastdfs](https://github.com/happyfish100/fastdfs)

[go-fastdfs](https://github.com/sjqzhang/go-fastdfs)

[seaweedfs](https://github.com/chrislusf/seaweedfs)

[gin-vue-admin](https://github.com/flipped-aurora/gin-vue-admin)