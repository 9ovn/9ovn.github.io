# 如何构建博客

![background](/blog/github-actions-hugo/githu-actions-hugo.jpg "GitHub  Actions")

## 前言

如何用hugo搭建自己的博客，这里我就不展开说明，大家可以去B站或者随意百度“如何使用hugo构建博客”，有大量的博文告诉你如何使用hugo。

假如你按照网络上的教程，成功的发布了自己的博客(这里默认你用的GitHub Pages)。那么你可能在想，如果我每写一篇博客是不是都要敲一下命令？

```shell
 hugo --theme="LoveIt" --baseUrl="https://****.github.io"  --buildDrafts
 cd ./public
 git add .
 git push
```

hugo有没有提供什么接口可以快捷的发送博文吗？很遗憾，hugo并不支持这一功能。不过，还好有GitHub Action。借助github这一功能，我们能够解决这个问题，自动发布部署我们的博客。(实际上我觉得并没有太多工作，但是还是想折腾一下:laughing:)

## 啥是GitHub Actions？

GitHub Actions 是GitHub在2018年出的一个CI/CD
服务。这里就不展开CI/CD是什么了，后期会出一系列的CI/CD的博客。简单的理解就是提交代码到GitHub，GitHub会自动将你的代码部署到服务器中，在本文中就是将hugo源码自动编译然后部署到你的GitHub Pages中。

## 使用Github Actions部署博客

### Step 1: 创建一个名字为<username>.github.io

创建一个名字为<username>.github.io的仓库，这个仓库用来存放编译后的Public文件，使用使用hugo编译时指定的url，就可以访问你的博客了。

### Step 2: 创建另外一个仓库用来存放你的Hugo源码

创建一个仓库用来存放你博客的源码，命名没有什么特别的规则，看自己喜好。需要注意的一点是，如果使用了hugo主题那么最好用git submode的方式引入到你的源码中，并且再git push之前确保更新主题到最新版本。

```shell
git submodule update --init --recursive
git push origin master
```

当然这一步我们也可以借助actions来帮助我们来进行这一步操作。

### Step 3：创建GitHub Token

接下来我们需要在[Github Token](https://github.com/settings/tokens/new)页面生成一个Token用于Actions中连接到你的仓库，拉取代码，更新主题，并将生成的pubic文件推送到你的<username>.github.io仓库中。
![token](/blog/github-actions-hugo/token.jpg "GitHub Token")

### Step 4: 添加token

将上一步的token放入hugo源码仓库的setting中的secrets中，
![setting](/blog/github-actions-hugo/secrets.jpg "setting ")
![add-token](/blog/github-actions-hugo/add_secrets.jpg "add token")

### Step 5: **编写GitHub Actions**

```yaml
name: ci-cd
on: push
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Git checkout
        uses: actions/checkout@v2.3.4
        with:
          submodules: recursive
          token: ${{ secrets.TOKEN}}
      - name: Update theme
        # 这里自动更新主题
        run: git submodule update --init --recursive

      - name: Setup hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: "latest"

      - name: Build
        # minify不需要可以移除
        # docs: https://gohugo.io/hugo-pipes/minification/
        run: hugo --minify --buildDrafts

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          personal_token: ${{ secrets.TOKEN }}
          external_repository: <username>.github.io
          publish_dir: ./public
          user_name: username
          user_email: email@***.com
          publish_branch: main # 自定义
          # cname参数填写你的自定义域名，可有可无
          # cname: example.com

```

在源码目录中创建.github/workflow文件，然后创建一个yml文件，将上述的代码贴入yml文件中，修改信息后就欧克了。

当你像源码仓库推送文件时，自动触发这个名字为ci-cd的actions。我们一步步看这个actions都做了什么动作

1. ***deploy.runs-on*** 声名了整个流程基于ubuntu-latest运行
2. ***Git checkout*** 这一步会获取到源码。with.submodules会将主题的代码拉下来，with.token会获取到我们之前设置的token用于连接仓库。
3. ***Update Theme*** 这一步会更新主题的代码。
4. ***Setup hugo*** 会使用actions-hugo来获取hugo需要的运行环境。
5. ***Build*** 构建博客。
6. ***Deploy*** 使用actions-gh-pages来将生成的pubilc文件，推送到你的github page仓库。

### Step 6：将源码push到github中

我们将源码提交到GitHub中，然后actions就自动帮我们部署页面了！
![actions-status](/blog/github-actions-hugo/actions-status.jpg "actions界面详情")
在这里我们可以点击到对应的deploy中查看部署详情。

等待actions部署成功后，等待几分钟就去个人静态页面中查看自己的[博客](http://9ovn.github.io)了！

## end

整个部署过程还是比较简单明了的，不过遗憾的是我也刚接触hugo和actions两个不久，不过暂时我也不准备深入了解这两个工具。目前会的这些东西也能够支撑我写博客了，边用边学吧！88:smirk:

