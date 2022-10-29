# 如何在多台设备上维护自己的 hexo 博客

### 准备工作

首先, `hexo` 可以抽象理解为由前后端组成

1. hexo 本身的运行环境:包括 hexo 的配置,hexo 主题的配置 (后端)
2. hexo 生成的博客的部分 (前端)

我们实际看到的博客页面是 `hexo` 的 **前端** 部分,其内容是根据我们在 `hexo` 创作时,由 **后端** 生成的

所以可以简单得出以下结论:
1. hexo 的运行环境(后端)部分是相对稳定的,不容易发生改变的,仅仅在对 hexo 的配置做出修改时才会有改动
2. 前端博客是经常变动的,但是它是由后端生成的

简单来说,我们通过在 `github` 仓库里,使用两个分支来分别维护这两部分代码即可

这里约定:
```
hexo 后端部分 ->>> hexo 分支维护
hexo 博客部分 ->>> master 分支维护
```

### 1. github page 的配置

`github page` 要把构建的分支设置为 `hexo` 的前端分支,也就是刚刚提到的 `master` 分支

![1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m7p8lcmaj31ny0o8wo3.jpg)

接着需要把默认分支设置为 `hexo` 的后端分支,也就是刚刚提到的 `hexo` 分支

![2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m7qcoo7aj31qc0hen5j.jpg)

### 2. hexo 准备工作

这里以全新的博客举例说明(如果旧电脑已有博客,则需要迁移 `hexo` 的后端代码)

由于是全新的博客,自然 `github` 仓库里也没有任何代码,直接将其 `clone` 到本地

因为之前设置了默认分支为 `hexo` 分支,所以我们需要在 `hexo` 分支上创建运行环境,直接运行以下命令

```bash
hexo init
```

这会在 `hexo` 分支上初始化一个全新的 `hexo` 博客后端环境

**如果需要迁移旧的博客怎么办: 把原来旧的博客目录下的文件夹挨个复制到当前目录,注意去掉 .git 和 .deploy_git 这两个旧的目录,其他目录组织按照 init 生成的对照起来**

接着修改 `hexo` 的部署设置,编辑 `_config.yml` 文件,填入 `github` 的仓库地址

注意将 `push` 分支设置为 `master` 即我们 `hexo` 的远程部分使用 `master` 管理维护

```yml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: 'git'
  repo: git@github.com:n1ck3dcydoom/n1ck3dcydoom.github.io.git
  branch: master
```

这样无论在什么分支编辑 `hexo` 的运行环境代码,只要这个配置文件没有发生改动, `hexo` 部署的时候都会往 `master` 推送

### 3. hexo 博客编写

在 `hexo` 分支上,完成博客创作后(具体指令不在这次讨论范围之内),直接执行 `hexo` 的部署命令,将生成的前端代码,推送到 `master` 分支上

```bash
hexo d
```

### 4. 其他设备怎么共同维护 hexo 博客

1. `clone` 远程的博客仓库,由于默认分支被设置为 `hexo` 那么拉下来的代码就是 `hexo` 的运行环境

2. 安装 `npm` 环境(若有则跳过),安装 `hexo`(若有则跳过), 执行 `npm install` 安装当前 `hexo` 的环境,大部分情况下只需要执行 `npm install` 就够了

3. 在 `hexo` 分支上完成博客创作,并将博客部署到远程 `hexo d`

4. 由于 `_config.yml` 的配置,其默认的部署分支就是 `master` 

这样,就完成了博客在多台设备上的共同维护