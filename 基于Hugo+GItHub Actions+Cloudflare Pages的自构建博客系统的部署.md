## 前言

Hexo，Hugo 相较于 Wordpress，Typecho 等一众以 PHP 为基础的 CMS 系统而言，存在内容发布困难的缺点。Typecho 发布新内容只需在后台编辑好内容点击发布即可，而 Hugo 等静态站点生成器需要将内容以 Markdown 格式放入指定文件夹，手动执行命令生成静态站点，再通过其他方式上传至网站目录，过程极为繁琐，且对多客户端用户不是很友善。本文提供一种基于 Hugo+GItHub Actions+Cloudflare Pages 的博客部署方式，能够将该过程大幅度简化。

## 准备工作

1. 三个 GitHub 仓库
   - Articles 仓库，用于储存 Markdown 格式的文章
   - Hugo 仓库，用于储存生成站点所需的 Hugo 配置文件与主题
   - Blog 仓库，用于储存生成的静态站点文件

2. 配置好的博客，本文不会介绍如何配置 Hugo，请自行寻找网络教程

## 配置 Hugo 仓库

1. 在 Hugo 文件夹内新建`.gitignore`文件，并粘贴如下内容

```
/public/
/content/posts/
/themes/terminal/resources/
/resources/
```

2. 删除主题下的`.git`文件夹，或者将主题添加为 Git Submodule
3. 用你精湛的 Git 技术把 Hugo 文件夹内的文件上传到 Hugo 仓库，这里提供一份命令参考
```
git init
git branch -M main
git add .
git commit -m "Initial Commit"
git remote add origin YOUR_REPOSITORY
git push -u origin
```

## 配置 Articles 仓库

1. 打开 https://github.com/settings/tokens ，点击 Generate new token -> Generate new token(classic) ，在 Select scopes中 勾选 repo 选项，并记住生成的 Token （Token只会显示一次，请妥善保管！）

2. 打开 Blog 仓库，点击 Settings -> Secrets -> Actions -> New repository secret ，Name 填写 TOKEN， Secret 填写在第一步保存的 Token

3. 在 Blog 仓库内点击 Actions -> set up a workflow yourself ，在编辑框内粘贴如下内容（部分内容需要更改！）

   ```yaml
   name: CI
   
   on:
     push:
       branches: [ "main" ]
     workflow_dispatch:
   
   jobs:
     build:
       runs-on: ubuntu-latest
   
       steps:
         - uses: actions/checkout@v3
   
         - name: Install Hugo
           run: |
             sudo apt update -y
             wget https://github.com/gohugoio/hugo/releases/download/v0.105.0/hugo_0.105.0_linux-amd64.deb
             wget https://github.com/gohugoio/hugo/releases/download/v0.105.0/hugo_extended_0.105.0_linux-amd64.deb
             sudo dpkg -i hugo_0.105.0_linux-amd64.deb 
             sudo dpkg -i hugo_extended_0.105.0_linux-amd64.deb
             
         - name: Preprocess Articles
           run: |
             git clone https://${{secrets.TOKEN}}@github.com/<CHANGE IT TO YOUR HUGO REPOSITORY URI!>
             for i in *.md; do file_basename=$(basename "$i" .md); file_date=$(date -d @$(stat -c %W "$i") --rfc-3339=seconds | sed 's/ /T/'); metadata="---\ntitle: $file_basename\ndate: $file_date\n---"; firstline=$(head -n 1 "$i"); if [[ $firstline != -* ]]; then sed -i "1 i $metadata" "$i"; fi; done
             mkdir <CHANGE IT TO YOUR HUGO REPOSITORY NAME!>/content/posts
             cp *.md <CHANGE IT TO YOUR HUGO REPOSITORY NAME!>/content/posts
           
         - name: Generate Site
           run: |
             cd wol-blog-hugo
             hugo -D
         
         - name: Publish
           run: |
             cd wol-blog-hugo/public
             git config --global user.email "<CHANGE IT TO YOUR EMAIL!>"
             git config --global user.name "<CHANGE IT TO YOUR NAME!>"
             git config --global init.defaultBranch main
             git init
             git branch -M main
             git add .
             git commit -m "Update Site at $(date "+%Y%m%d-%H%M%S")"
             git remote add origin https://${{secrets.TOKEN}}@github.com/<CHANGE IT TO YOUR BLOG REPOSITORY URI!>
             git push --force -u origin main
   ```

   `<CHANGE IT TO YOUR BLOG REPOSITORY URI!>` 替换为 `用户名/仓库名.git` 的形式，例如 `WOL/WOL-HUGO.git`
   
   `<CHANGE IT TO YOUR HUGO REPOSITORY NAME!>` 替换为仓库名，例如 `WOL-HUGO`
   
   `<CHANGE IT TO YOUR EMAIL!>` `<CHANGE IT TO YOUR NAME!>` 分别替换为Git应使用的邮箱与名字
   
   点击 Commit ， Actions 将自动运行，若运行成功即代表配置文件正确，此时可以打开 Blog 仓库检查生成的静态站点文件
   
   **What Happened ?**
   
   这个配置文件中最难理解的或许是 `for i in *.md; do file_basename=$(basename "$i" .md); file_date=$(date -d @$(stat -c %W "$i") --rfc-3339=seconds | sed 's/ /T/'); metadata="---\ntitle: $file_basename\ndate: $file_date\n---"; firstline=$(head -n 1 "$i"); if [[ $firstline != -* ]]; then sed -i "1 i $metadata" "$i"; fi; done` 一行，这实际上是一个压缩成一行的Bash脚本，展开后是这个样子：
   
   ```bash
   for i in *.md;
    do
        file_basename=$(basename "$i" .md);
        file_date=$(date -d @$(stat -c %W "$i") --rfc-3339=seconds | sed 's/ /T/');
        metadata="---\ntitle: $file_basename\ndate: $file_date\n---";
        firstline=$(head -n 1 "$i");
        if [[ $firstline != -* ]];
        then
            sed -i "1 i $metadata" "$i";
        fi;
    done
   ```

　这是一个自动生成 Metadata 的 Bash 脚本，带有对自定义 Metadata 的检测功能。

   其中，`$(basename "$i" .md)` 表示使用文件名作为文章的标题， `$(date -d @$(stat -c %w "$i") --rfc-3339=seconds | sed 's/ /T/')` 表示以文件创建日期作为文章的创建日期（TOML使用RFC-3339格式表示时间）

## 配置 Cloudflare Pages

> 也可以使用 Vercel ，Github Pages 等静态站点部署服务

1. 打开 https://dash.cloudflare.com/ ，点击 Pages -> 创建项目 -> 连接到Git -> 添加账户，授权 Cloudflare Pages 应用访问 Blog 仓库，点击 开始设置
2. 在 设置构建和部署 页面修改项目名称（可选），无需更改下列设置，直接点击 保存并部署
3. 等待部署完成后，访问Cloudflare提供的 pages.dev 域名，即可看到部署好的博客

## Have a try

打开 Articles 仓库，上传一篇 Markdown 文章，等待构建完成后打开博客即可看到上传的文章

## It‘s not the end

这套部署方案相比于手动部署而言极大程度降低了工作量，但其仍然存在不足：

1. 将Hugo的安装文件链接硬编码在了配置文件中
2. 需要手动上传 Markdown 文件，较为繁琐

简而言之，就是仍有改进空间，敬请期待
