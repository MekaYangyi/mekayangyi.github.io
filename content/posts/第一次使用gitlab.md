---
title: 第一次使用gitlab
date: 2017-01-28 09:19:43
categories: 
- gitlab
tags:
- gitlab
---

# 安装git
版本差别不大，目前使用的版本git2.11.0.3。
一路下一步,不修改安装位置，直接使用默认设置。
# 安装TortoiseGit
一路下一步，不修改安装位置，直接使用默认设置。
# 登录账户
## 管理员创建账户
Admin→New User。新建账户、并设置密码。同时设置该用户所属Group。
> name：上传显示的名字，可以经常更改，使用中文名好。

> username：登陆的用户名，不可修改，用于账户登陆。

> email：账户email，内网联系email。

> password：密码，牢记，root用户可修改。

## 登录
登陆内网gitlab，目前网址：10.10.10.98。
<img src="/img/第一次使用gitlab/22-04-07.jpg" alt=""/>

<!--more-->
## 修改密码
登录后会要求修改密码，自己输入即可。
<img src="/img/第一次使用gitlab/22-05-00.jpg" alt=""/>
# 设置TortoiseGit
将注册的用户填入TortoiseGit中,设置为全局账号。
<img src="/img/第一次使用gitlab/21-07-01.jpg" alt=""/>
# 添加ssh
如果不添加shh，每次修改都会要求输入账号密码，比较麻烦。添加后与设备绑定，修改不再需要填写账号密码。如果不再使用一个设备，请删除ssh。
## 生成ssh
任意空白处，选择git bash。
<img src="/img/第一次使用gitlab/06-27-40.jpg" alt=""/>
窗口中输入，一路按回车。
```
ssh-keygen -t rsa -C "你注册的email地址"
```
生成后的公钥会存放在 C:/Users/You_User_Name/.ssh/id_rsa.pub。
用记事本打开，复制。
<img src="/img/第一次使用gitlab/06-30-35.jpg" alt=""/>
## 将ssh加入gitlab
浏览器转到http://10.10.10.98/profile/keys。
将复制的Key粘贴，Add key。
<img src="/img/第一次使用gitlab/06-32-52.jpg" alt=""/>
# 新建一个项目
## 新建一个私有项目
新建一个私有项目进行实验，gitlab是否能够正常上传。
选择new project。
<img src="/img/第一次使用gitlab/06-07-09.jpg" alt=""/>
## 填写基本信息
> Project name：项目名称。
> Project description：项目描述。
> Visibility Level：项目级别，内网使用只选择public与private。private除了项目成员不可见，public在内网均可见。

选择Create project创建项目。
<img src="/img/第一次使用gitlab/06-08-43.jpg" alt=""/>
# Clone项目 
项目创建完成，复制项目地址。
复制浏览器的项目地址：http://10.10.10.98/test/test_project。
<img src="/img/第一次使用gitlab/06-14-02.jpg" alt=""/>
选择计算机一个文件夹，右键，选择git clone。
<img src="/img/第一次使用gitlab/06-15-28.jpg" alt=""/>
clone之前复制地址项目，确认，输入用户名与密码，clone成功，会新建一个文件夹。
<img src="/img/第一次使用gitlab/06-16-37.jpg" alt=""/>
# 第一次push
在该文件夹中建立一个README.md文件。
<img src="/img/第一次使用gitlab/06-18-54.jpg" alt=""/>
## add
右键add，确认，ok。
<img src="/img/第一次使用gitlab/06-19-25.jpg" alt=""/>
## commit
commit将修改更新到本地。
在项目文件夹空白处右键，选择git commit。
<img src="/img/第一次使用gitlab/06-25-26.jpg" alt=""/>
填写上传理由，commit。
<img src="/img/第一次使用gitlab/06-26-03.jpg" alt=""/>
## push 
选择push，push能够将修改push到服务器。
<img src="/img/第一次使用gitlab/06-36-23.jpg" alt=""/>
确认。
<img src="/img/第一次使用gitlab/06-37-12.jpg" alt=""/>
成功界面。
<img src="/img/第一次使用gitlab/06-37-36.jpg" alt=""/>
# 查看结果
成功在网页端看到修改记录(之前误上传为.md.txt文件，修改为.md文件网页端会显示文字)。
<img src="/img/第一次使用gitlab/06-40-03.jpg" alt=""/>