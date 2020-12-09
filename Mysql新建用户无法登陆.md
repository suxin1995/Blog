	重新搭建Mysql环境后，root账号登陆正常，而重新创建的新用户 'test‘始终无法正常登陆到数据库中。

```shell
mysql -u test -p
password:
```

​	登陆失败，提示报错：

ERROR 1045 (28000): Access denied for user 'test'@'localhost' (using password: YES)

​	尝试登陆root账号并为新用户授权，依然无法解决。

​	最后发现，Mysql默认存在用户名为空的账户，在本地可不用输入账号密码即可登陆。由于该特殊账户的存在，使其他用户无法正常登陆。

##### 解决方案：

​	登陆root账户后，进入user 库，删除无效账号即可。

```shell
use mysql   #选择mysql库

delete from user where User='';  #删除账号为空的行

#授权用户外联权限
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'mypwd' WITH GRANT OPTION;

flush privileges;  #刷新权限
exit  #退出mysql
```

​	验证尝试登陆成功



