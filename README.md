# 网页平台相关操作记录

## CentOS服务器配置相关

### 1. 网站服务器的开启（使用Linux+Nginx+MySQL+PHP即lnmp）
  1. lnmp官网装lnmp环境
  2. 更改nginx配置：`vim /usr/local/nginx/conf/nginx.conf`
  3. 把server大括号内里的默认网页根目录改成想要的根目录
  4. 把网页拷进新的目录：`chmod -R 777 目录（如果涉及到上传文件的功能，必须赋予所有用户写入权限）`

### 2. 更改跨域访问设置
  本地写的vue项目想要访问服务器数据，要设置允许跨域访问（CORS）。如果使用nginx代理（将请求通过nginx转发到api服务端口），就不需要跨域。
  1. nginx.conf下的server括号下加入以下内容：
  ```
  add_header 'Access-Control-Allow-Methods' 'GET,OPTIONS,POST' always;
  add_header 'Access-Control-Allow-Credentials' 'true' always;
  add_header 'Access-Control-Allow-Origin' $http_origin always;
  add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, X-Requested-With, Cache-Control' always;
  if ($request_method = OPTIONS ) { return 200; }
  ```

  2. 重启ngxin：`service nginx restart`

### 3. 开放端口方法
  能通过nginx转发就尽量不要直接对外开放，不安全
  1. 修改本机防火墙策略表：`vim /etc/sysconfig/iptables`
  2. 插入：`-A INPUT -p tcp -m tcp --dport 1234 -j ACCEPT`（针对需要全部开放的端口如http(s)端口80和443）或`-A INPUT -p tcp -m state --state NEW -m tcp --dport 3306 -j ACCEPT`（针对需要安全的端口，如MySQL的3306）
  3. 重启防火墙：`service iptables restart`
  4. 开启云服务器供应商的防火墙

### 4. Nginx设置虚拟目录托管静态文件
  比如在浏览器输入：`http://ip地址/static/xxx.jpg`，nginx自动代理到资源文件夹下
  1. 在nginx.conf中添加location设置：
  ```
  location ~ /geotiff/ {
      root  /home/Website/;
      #或者alias /home/Website/geotiff也可以
      # autoindex on;#开启文件目录形式
      }
  ```
  2. 如果是远程调试，要记得加上跨域访问请求头
  3. 给目录添加权限：`chmod -R 777 /home/Website/geotiff/`
  4. 重启nginx

### 5. Nginx托管api接口
  比如在浏览器输入：`http://ip地址/api/xxx`，nginx自动代理到api服务端口`http://ip地址:3000/api/xxx`
  1. 在nginx.conf中添加location设置：
  ```
  location ~ /api/ {
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      #proxy_set_header X-Nginx-Proxy true;
      proxy_set_header Connection "";
      proxy_pass http://127.0.0.1:3000;
  }
  ```
  2. 重启nginx

### 6. Nginx托管websocket接口
  比如前端使用websocket服务：`ws://ip地址/ws/`，nginx自动代理到ws服务端口`http://ip地址:8082/`
  1. 在nginx.conf中添加http设置：
  ```
  #My WebSocket Proxy Config
  map $http_upgrade $connection_upgrade {
      default upgrade;
      '' close;
  }
  ```
  2. 添加server设置：
  ```
  location /ws/ {
      proxy_pass http://127.0.0.1:8082;
      proxy_http_version 1.1;
      #以下配置添加代理头部：
      proxy_set_header Host $host; # 保留源信息
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
  }
  ```
  3. 重启nginx

## 数据库相关

### 1. 数据库忘记密码
  版本：MySQL 5.7

  1. 关闭数据库：`service mysql stop`
  2. 找到MySQL的位置文件：`find / -name my.cnf`
  3. 添加一行`skip_grant_tables`，跳过数据库登录密码验证
  4. 启动数据库：`service mysql start`
  5. 直接输入`mysql`登录数据库
  6. 重设root用户：
  ```
  UPDATE mysql.user
  SET authentication_string=PASSWORD('newp_assword')
  WHERE user='root' AND host='localhost';

  FLUSH PRIVILEGES;
  ```
  7. 删除第3步添加的`skip_grant_tables`
  8. 重启数据库`service mysql restart`

### 2. 数据库创建只读用户
  因为给root用户设置远程权限太危险，所以最好创建一个只能读写的用户进行远程连接。
  ```
  GRANT CREATE,SELECT,INSERT ON your_data_base.* TO username@"%" IDENTIFIED BY "password";

  flush privileges;
  ```

### 3. 建表的SQL语句
  以上传风速测量数据为例：
  ```
  CREATE TABLE IF NOT EXISTS `zb_wind_data`(
    `id` INT UNSIGNED AUTO_INCREMENT,
    `date` DATE,
    `time` TIME,
    `70m_v_avg` FLOAT,
    `70m_v_max` FLOAT,
    `70m_v_min` FLOAT,
    `60m_v_avg` FLOAT,
    `60m_v_max` FLOAT,
    `60m_v_min` FLOAT,
    `70m_deg_avg` FLOAT,
    `70m_deg_max` FLOAT,
    `70m_deg_min` FLOAT,
    `40m_deg_avg` FLOAT,
    `40m_deg_max` FLOAT,
    `40m_deg_min` FLOAT,
    PRIMARY KEY ( `id` )
  )ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```

### 4. 查询相关的SQL语句
  以上面的风速测量数据为例，记录在[这里](./content/SQL.md)。（仅参考，要根据实际的数据表结构考虑）

## 服务器后端api的开发

### 1. 常用框架：
  - CakePHP (PHP语言)
  - Django (python 适合大型项目)
  - Flask (python 适合小型项目)
  - Express (javascript)
  - Spring Boot (Java)
  - ...
  
### 2. Express框架搭建
  1. 服务器安装node.js和npm（不要直接用apt-get或yum，安装，要去官网下载）
  2. 创建项目文件夹，`npm init`初始化项目，一路回车
  3. 按照Express教程引入Express包开始开发
  4. 更详细参考[B站教程](https://www.bilibili.com/video/BV1mQ4y1C7Cn/?spm_id_from=333.337.search-card.all.click)
  5. 可以通过Nginx的代理功能把`http://your.server.ip.address:xxxx/`的请求代理到`http://your.server.ip.address/api/`下，就避免了端口暴露的隐患。
  6. 调试接口的工具：Apifox、postman，用来做接口管理很方便

### 3. WebSocekt接口实现
  1. 和Express一样，在node.js下引入nodejs-websocket包
  2. 网上很多简单的起步教程
  3. 前端页面通过`ws://your.server.ip.address:xxxx/`就能和服务器通信，也可以像Express一样通过nginx代理，但是ws协议需要通过nginx升级，具体查阅相关资料

### 4. 服务如何常驻、终止
  1. 通过screen指令可以让服务在后端长时间运行：
  ```
  screen -S name #以name为代号开一个进程
  screen -r name #重新打开关闭的进程
  ```
  2. 或者通过nohup指令后台运行
  3. 终止后台运行的nodemon进程
  ```
  ps -ef | grep nodemon #查找进程
  kill 12345 #杀掉进程
  ```

## Git相关

### 1. Github被墙时，命令行Git提交的方法
  就算挂了梯子，用命令行提交到git的时候也没有用，需要设置命令行代理：
  1. 挂梯子
  2. 设置命令行代理：
  ```
  #端口号11223是梯子的端口号
  git config --global http.proxy http://127.0.0.1:11223
  git config --global https.proxy http://127.0.0.1:11223

  #下两行可选，我也不知道是什么
  git config --global http.proxy 'socks5://127.0.0.1:11223'
  git config --global https.proxy 'socks5://127.0.0.1:11223'

  # 取消代理
  git config --global --unset http.proxy
  git config --global --unset https.proxy
  ```
   