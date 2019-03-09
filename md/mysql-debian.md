# MYSQL Debian的安装

## 更新apt-get仓库
1. 前往官网[下载](https://dev.mysql.com/downloads/repo/apt/)相关的deb。
2. 使用dpkg -i [下载的文件名] 安装deb。并使用以下命令更新。
   ```
   apt-get update
   ```
3. 使用以下命令安装
```
apt-get install mysql-server
```
其他命令：
```
service mysql status 查看服务状态
service mysql stop 停止服务
service mysql start 启动服务
```