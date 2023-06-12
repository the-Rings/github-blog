---
title: 使用Hexo构建博客
date: 2021-05-28 14:53:12
categories:
- Hexo
---

一直以来，使用其他工具来记录技术知识的积累，以后陆续将其转到github上。使用Hexo搭建项目可以自动将文档生成静态HTML等文件，上传到github上，随时通过your-github-name.github.io查看自己的公开静态信息库。

## Github准备操作

1. 首先在github上创建一个新的repository，其中`Repository name`为`[your-github-name].github.io`，设为开源, 用来存放生成的静态文件, 先不创建任何文件和分支.
2. 在创建一个repository, `Repository name`最好设为`github-bolg`(根据个人喜好), 这个项目存放文档以及源码
3. 将ssh公钥`id_ras.pub`内容添加到个人信息 > settings > SSH keys

## 本地准备操作

1. 安装Node.js (默认安装了npm)

2. npm的配置, 对前端不是很熟, 这里需要记录一下
   - 在Windows默认的用户目录下, 新建`C:\Users\your-user-name\.npmrc`文件
     ```
       registry=https://registry.npm.taobao.org
       prefix=D:\your-path\nodejs\node_global
       cache=D:\your-path\nodejs\node_cache
    ```

3. 安装`npm install -g hexo`

4. 初始化`hexo init [your-project-name]`

5. 此时可以进行一步本地启动尝试. 执行三个命令: `hexo clean`&`hexo generate`&`hexo server`, 如果成功访问`localhost:4000`说明初始化成功

6. 进入项目根目录查看

   - `.deploy_git`&`.github`不明其意, 勿动
   - `source/_posts`存放源文档, 未来soure中还会加入`about`, `categoryies`, `tags`文件夹, 分别对应了`关于` , `分类`, `标签`页面的源文档. 
   - `themes`对应主题, 每个主题都会是一个文件夹
   - `_config.yml`记录了整个项目的配置, 包括显示的语言, 时区等
   - `package.json`&`package-lock.json`为项目依赖包的记录
   - `public`文件夹是生成的静态文件保存的位置, 对应`[your-github-name].github.io`仓库
   - `node_modules`对应依赖包信息

7. 整个项目初始化为git项目, 并添加`.gitignore`文件

    ```
     .DS_Store
     Thumbs.db
     db.json
     *.log
     node_modules/
     public/
     .deploy*/
    ```

   - > 通过此gitignore文件可以看出向`[github-blog]`repository推送的包括源文档, 主题文件和配置文件

     ```shell
     git init
     git remote add origin [github-bolg repository ssh address]
     # for example: $ git remote add origin git@github.com:the-Rings/github-blog.git
     # github从2020年12月份之后, 不再支持用户名&密码方式推送项目, 所以这里是ssh address, 配合我们之前加入的SSH keys使用
     git add .
     git commit -m "something"
     git push --set-upstream main
     # 推送完成
     ```

8. 配置`_config.yml`向`[your-github-name].github.io`仓库推送public下的文件, 这是不需要初始化git仓库, 使用hexo相关工具完成

    ```yaml
    # 编辑_config.yml中的deploy选项
    deploy:
      type: git
      repo: git@github.com:[your-github-name]/[your-github-name].github.io.git # 复制仓库的SSH地址
      branch: main # 分支默认为main即可
    ```

9. 安装`npm install hexo-deployer-git --save`, 这个工具来向github推送静态文件

10. 通过`hexo clean`&`hexo generate`&`hexo deploy`三个命令组合完成, 实际上每次修改之后都是这一顿操作.

## 主题等相关配置操作

1. 选用`Next`主题, 然后对`theme/next`中的配置文件进行相关设置, 这里参考网友们的答案即可, 主要是对首页、归档、分类、标签、关于进行相关设置和注释放开, 并通过命令增加对应的模块

   ```shell
     hexo new page categories
     hexo new page tags
     hexo new page about
   ```

2. 启用搜索功能, 在项目的根目录下`_config.yml`找到`Extensions`, 增加配置, 并执行`$ npm install hexo-generator-searchdb --save`

    ```yaml
     search:
       path: search.xml
       field: post
       format: html
       limit: 10000
    ```

   - 然后, 在主题配置文件`theme/next/.config.yml`中设置`local_search: enable: true`

3. UML图插件
    ```shell
    npm install --save hexo-tag-plantuml
    ```
    语法参考: https://plantuml.com/zh/class-diagram
    预览效果: https://www.plantuml.com/plantuml/uml/SyfFKj2rKt3CoKnELR1Io4ZDoSa70000
    {% plantuml %}
    Joinpoint <|-- Invocation
    interface Joinpoint {}
    interface Invocation {}
    
    Invocation <|-- MethodInvocation
    interface MethodInvocation {}
    
    MethodInvocation <|-- ProxyMethodInvocation
    interface ProxyMethodInvocation {}
    
    ProxyMethodInvocation <|.. ReflectiveMethodInvocation
    {% endplantuml %}

## 插入图片
下载插件
```shell
npm install hexo-renderer-marked --save
```
在`_config.yml`做如下更改
```yaml
post_asset_folder: true
marked:
  prependRoot: true
  postAsset: true
```
这样每次`hexo new post "xxx"`后，就会在`_post`下生成一个同名文件夹，将图片放在这个同名文件夹中即可，然后在引用图片的地方加上`![](image_name.jpeg)`即可

## 特殊语法
1. 站内文章链接{% post_link file_name "display name" %}

## 在一台新的机器上克隆项目的流程
```shell
git clone your-github-blog-url
cd your-github-blog-root-path
# set your-node-module-path to PATH
npm install -g hexo
# 从package-lock.json安装所有包
npm ci
hexo clean
# 如果出现NEXT主题的logo说明安装成功
# 如果出现“hexo: 无法加载文件 ******.ps1”的错误，可能要以管理员身份运行cmd，“set-ExecutionPolicy RemoteSigned”
```
#### 日常操作流程
1. 写博客`hexo new post [new-blog-name]`, 在`source/_post/`下就生成了一个新的博客文件, 可以用其他markdown编辑工具来编写, 比如Typora
2. 在博客文档的开头可以设置标签和分类
3. 写完之后, 提交源码`git add `&`git commit`&`git push`
4. 部署发布`hexo clean`&`hexo generate`&`hexo deploy`



# Troubleshooting
总结一些在部署中的错误解决
1. 在执行`hexo generate`中出现错误: `Template render error: (unknown path)`, 多半是语法错误, 比如, 在使用`{% xxx %}`时写成了`{ % xxx %}`, 前方多了一个空格.