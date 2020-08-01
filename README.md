## 用 Docker 搭建 vue 应用

### 说在前面
网上很多docker部署vue项目的教程，其中很多的文章不乏都是先将vue项目执行npm run build 在本地进行打包，传到自己的仓库去，然后到服务器去拉取我们的代码，获取dist文件，再将该文件挂载到dockr容器内。其实这种操作应当是有缺陷的，我们应当把打包的操作也放到docker的镜像里面去操作。

### 准备好vue项目
```
vue create vueTest
```

### 创建 Dockerfile 文件

```
FROM node:12.16.3-alpine AS builder

# 将容器的工作目录设置为/app(当前目录，如果/app不存在，WORKDIR会创建/app文件夹)
WORKDIR /app 

COPY ./package*.json /app/ 

#安装依赖
RUN npm config set registry "https://registry.npm.taobao.org/" \
  && npm install 
COPY . /app
RUN npm run build 


#指定nginx配置项目，--from=builder 指的是从上一次 build 的结果中提取了编译结果
FROM nginx

#将打包后的文件复制到nginx中
COPY --from=builder app/dist /usr/share/nginx/html/

#用本地的 default.conf 配置来替换nginx镜像里的默认配置。
COPY --from=builder app/nginx.conf /etc/nginx/conf.d/default.conf

#暴露容器80端口
EXPOSE 80

```

可以看到，在这里将打包操作也放到Dokcerfile里面进行操作了。


**该条命令是将我们在镜像里面打包生成的dist文件放进容器内nginx的web目录下面。**
```
COPY --from=builder app/dist /usr/share/nginx/html/
```

**该条命令是将我们项目目录下面的nginx.conf文件复制到容器内nginx的配置文件的目录下面，从而覆盖原有的配置文件。**
```
COPY --from=builder app/nginx.conf /etc/nginx/conf.d/
```

### 创建 nginx.conf 文件
在项目根目录下创建nginx文件，该文件夹下新建文件nginx.conf
```
server {
    listen       80;
    server_name  localhost;

    # 开启 gzip
    gzip  on;
    gzip_min_length 1k;
    gzip_buffers 16 64k;
    gzip_http_version 1.1;
    gzip_comp_level 9;
    gzip_types text/plain application/x-javascript application/javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary on;

    location / {
        # 根目录
        root   /usr/share/nginx/html;
        index  index.html index.htm;

        # 解决HTML5 History 模式
        try_files $uri $uri/ /index.html;
    }
}
```

### 创建 .dockerignore 文件
在项目根目录下创建.dockerignore，用与忽略镜像打包文件
```
 .git
 node_modules
 npm-debug.log
```

### 制作镜像
```
docker image build -t vuetest:1.0 .
```
-t 是给镜像命名 . 是基于当前目录的Dockerfile来构建镜像

### 启动容器
```
docker run \
-p 3000:80 \
-d --name vueTest \
vuetest
```

- `docker run` 基于镜像启动一个容器
- `-p 3000:80` 端口映射，将宿主的3000端口映射到容器的80端口
- `-d` 后台方式运行
- `--name` 容器名 查看 docker 进程

可以发现名为 vueTest的容器已经运行起来。此时访问 http://{ip}:3000 应该就能访问到该vue应用: