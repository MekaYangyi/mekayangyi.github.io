# gitlab workflow


建立一个长期分支，就是master，master分支上的版本都是能够编译运行的版本。
整个工作流程如下。
第一步：根据需求，从master拉出新分支，不区分功能分支或补丁分支。
第二步：新分支开发完成后，或者需要讨论的时候，先从master合并到分支，解决冲突，然后向master发起一个pull request（简称PR）。
第三步：Pull Request既是一个通知，让别人注意到你的请求，又是一种对话机制，大家一起评审和讨论你的代码。对话过程中，你还可以不断提交代码。
第四步：你的Pull Request被接受，合并进master，重新部署后，原来你拉出来的那个分支就被删除。（先部署再合并也可。）

# 建立测试项目
新建一个项目用于测试工作流。
演示项目地址：http://10.10.10.98/MekaYangyi/workflow
<img src="/img/gitlab工作流程/10-21-42.jpg" alt=""/>
# 设置分支保护
新建项目默认master用户才能够push和merge。
其余用户只能新建分支，在分支测试完毕后，再发起Merge Requests。发起后，由master进行审核后合并。
<img src="/img/gitlab工作流程/10-32-56.jpg" alt=""/>

<!--more-->

# 设置开发成员
项目创建者在项目页面选择Member。
<img src="/img/gitlab工作流程/10-31-10.jpg" alt=""/>

设置开发人员分为两种，一种是直接设置用户，一种是设置一Group都为指定权限。
权限分为四类：
> Guest
> Reporter
> Developer
> Master

一般开发人员指定为Developer。
具体权限在http://10.10.10.98/help/user/permissions.md查看。
设置用户权限
<img src="/img/gitlab工作流程/11-23-52.jpg" alt=""/>
设置整个Group的权限
<img src="/img/gitlab工作流程/11-24-20.jpg" alt=""/>
# 建立本地分支
在项目文件夹右键，选择TortoiseGit→Create Branch。
<img src="/img/gitlab工作流程/11-43-08.jpg" alt=""/>
填写信息
<img src="/img/gitlab工作流程/11-52-57.jpg" alt=""/>
# 切换分支
在项目文件夹右键，选择TortoiseGit→Switch/Checkout。
<img src="/img/gitlab工作流程/11-54-23.jpg" alt=""/>
选择OK。
<img src="/img/gitlab工作流程/11-55-03.jpg" alt=""/>
# Commit
分出分支后，可以在本地进行Commit，知道一个功能开发完毕后，再上传到服务器。
修改本地的文件，Commit。
<img src="/img/gitlab工作流程/11-57-19.jpg" alt=""/>
填写上传备注，Commit。
# 将Master向Test_WorkFlow合并
先从服务器pull最新版本，然后将master向Test_WorkFlow合并，防止master在分支分出之后被修改导致的冲突。
从服务器pull最新版本。
<img src="/img/gitlab工作流程/22-39-02.jpg" alt=""/>
将master向Test_WorkFlow合并。
<img src="/img/gitlab工作流程/22-41-09.jpg" alt=""/>
合并解决冲突。
<img src="/img/gitlab工作流程/22-41-37.jpg" alt=""/>
# push
上传成功后选择push。
<img src="/img/gitlab工作流程/22-44-54.jpg" alt=""/>

确认。
<img src="/img/gitlab工作流程/11-59-05.jpg" alt=""/>
成功。
<img src="/img/gitlab工作流程/11-59-45.jpg" alt=""/>
# 发起Merge Request
在项目页面找到分支，选择Merge Request
<img src="/img/gitlab工作流程/12-05-10.jpg" alt=""/>
填写相关信息，Submit merge request
<img src="/img/gitlab工作流程/12-06-31.jpg" alt=""/>
# 检视代码并讨论
选择Merge Request。
单击测试分支合并功能这一个Merge Request
<img src="/img/gitlab工作流程/12-08-38.jpg" alt=""/>
弹出页面。
<img src="/img/gitlab工作流程/12-09-43.jpg" alt=""/>
检视代码
## Commit，查看修改记录。
图中红色区域单击可以Diff与查看源文件。
<img src="/img/gitlab工作流程/12-11-44.jpg" alt=""/>
单击任意版本提交记录，增加检视意见，在diff时，任意处可以添加讨论。或者在页面底部对整个修改进行评价。
<img src="/img/gitlab工作流程/112-18-49.jpg" alt=""/>

<img src="/img/gitlab工作流程/12-19-25.jpg" alt=""/>

## Changes，查看版本区别。
<img src="/img/gitlab工作流程/12-12-47.jpg" alt=""/>
## Discusion，填写建议
填写建议后，选择Comment可以互相讨论。
或者选择Close merge request关闭请求。
<img src="/img/gitlab工作流程/12-15-41.jpg" alt=""/>
## Accept merge request或者Close merge request
<img src="/img/gitlab工作流程/12-22-44.jpg" alt=""/>
## 合并完成
选择Accept Merge Request,同时选择合并时将分支删除。
<img src="/img/gitlab工作流程/12-23-55.jpg" alt=""/>
合并结果。
<img src="/img/gitlab工作流程/12-25-09.jpg" alt=""/>
# 本地pull
在本地项目进行pull,同步服务器版本。
<img src="/img/gitlab工作流程/12-26-33.jpg" alt=""/>
本地同步结果。
<img src="/img/gitlab工作流程/12-27-07.jpg" alt=""/>
# 删除本地分支
由于远程分支与本地分支没有关系，那么当远程正式Merge之后，需要删除本地分支，防止以后分支一直增加，不减少。
在项目文件夹右键，选择TortoiseGit→Switch/Checkout，先把分支切换到本地master。
<img src="/img/gitlab工作流程/11-54-23.jpg" alt=""/>
切换完成后，再次进入该界面选择....
<img src="/img/gitlab工作流程/12-47-04.jpg" alt=""/>
删除分支
<img src="/img/gitlab工作流程/12-45-50.jpg" alt=""/>

# Issue
Issue 用于 Bug追踪和需求管理。建议先新建 Issue，再新建对应的功能分支。功能分支总是为了解决一个或多个 Issue。
## 新建Issue
选择New Issue
<img src="/img/gitlab工作流程/12-32-50.jpg" alt=""/>
填写相关信息。
功能分支的名称，可以与issue的名字保持一致，并且以issue的编号起首，比如"1-测试Issue"。
<img src="/img/gitlab工作流程/12-36-52.jpg" alt=""/>

新建完成,每一个Issue都有一个编号，本Issue的编号为#1。
<img src="/img/gitlab工作流程/12-37-36.jpg" alt=""/>
在分支开发完成后，在commit message里面，可以写上"fixes #14"或者"closes #67"。
<img src="/img/gitlab工作流程/12-39-48.jpg" alt=""/>
Github规定，只要commit message里面有下面这些动词 + 编号，就会关闭对应的issue。
> close
> closes
> closed
> fix
> fixes
> fixed
> resolve
> resolves
> resolved

这种方式还可以一次关闭多个issue，或者关闭其他代码库的issue，格式是username/repository#issue_number。
Pull Request被接受以后，issue关闭，原始分支就应该删除。如果以后该issue重新打开，新分支可以复用原来的名字。
## 查看结果
<img src="/img/gitlab工作流程/12-41-19.jpg" alt=""/>
<img src="/img/gitlab工作流程/12-41-51.jpg" alt=""/>
