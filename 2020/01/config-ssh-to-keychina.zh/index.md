# Config ssh to Keychina


## 将生成的ssh 私钥添加到 Mac 的keychain 中， 如果是其他操作系统，可忽略此步骤

```shell
$ ssh-add -K .ssh/is_rsa
```
## 将登录信息配置到.ssh/config中

```shell
$ touch ~/.ssh/config
$ vim ~/.ssh/config
# edit text
Host myvm
  Hostname ip
  User user
```

## 保存之后就可以使用如下命令快捷登录服务器了

```
$ ssh myvm
```

## 参考地址

<https://blog.infox.ren/2019/10/24/ssh-guide/>


----
![谷哥说-微信公众号](/images/wechat/扫码_搜索联合传播样式-标准色版.png)
