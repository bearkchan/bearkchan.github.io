---
title: "gitlab与gitflow结合使用流程"
date: 2020-03-02T00:29:44+08:00
draft: false
Categories: ["git"]
Tags: ["git", "版本控制", "gitlab"]
---

#### 1. 安装git-flow

打开mac终端，默认已安装`brew`，使用以下命令：

```shell
brew install git-flow
```

#### 2. gitlab上创建空项目

创建完项目后，添加项目成员，设置成员身份。

#### 3. git clone到本地

#### 4. 在本地使用git flow初始化分支

```shell
# 默认创建master分支和developer分支
git flow init
```

#### 5. 推送所有分支到gitlab

```shell
git push --all
```

#### 6. 项目管理者建立里程碑和issue

> 里程碑相当于一系列功能需求的集合，可以是[v1.0.0]等；

> issue相当于一个一个小的任务功能需求；

#### 7. 分配issue给对应的组员

#### 8. 组员根据对应的issue创建feature功能分支

```shell
git flow feature start question1
# 上述命令相当于以下命令：
git checkout -b feature/question1
```

#### 9. 提交到远程对应的分支上

```shell
# 提交commit信息的时候可以使用fix关键词来和issue关联
git commit -m "finish task,fix #1 ."

# 设置上游，这样就可以在下次直接git push 和 git clone了
git push --set-upstream origin local_branch_name



git push origin feature/question1
```

#### 10. gitlab上创建合并请求

> 注意：创建title的时候，可以使用`fixed`关键词加上issue对应的序号`#1`来和issue关联。

#### 11. 关闭对应的议题任务

```shell
git flow feature finish question1
```



## gitlab的定时备份

```shell
# 默认备份路径为：/var/opt/gitlab/backups/
gitlab-rake gitlab:backup:create 


# 定时备份
00 02 * * * gitlab-rake gitlab:backup:create &>/dev/null
```

