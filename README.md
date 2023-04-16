## 如何本地部署博客编写环境
1. 安装ruby。以Manjaro/Arch为例：
   ```bash
   sudo pacman -S ruby
   ```
2. 安装jekyll gems。gems可以认为是ruby的官方库，我们使用的jekyll框架就是一个gem。直接安装的方式如下：

   ```bash
   gem install jekyll
   ```

   不过我更推荐使用bundle进行gem包的管理：
    ```bash
    bundle init
    bundle config set --local path 'vendor/bundle'
    bundle add jekyll
    bundle install
    ```

3. 将github上的博客仓库直接clone到本地目录
4. 使用`jekyll serve`或者`bundle exec jekyll serve`即可完成渲染。打开`https://localhost:4000`就可以看到网站主页。如果你设置了`_config.yml`中的`baseurl`，那么站点的地址在`https://localhost:4000/baseurl`。

## 为什么选用jekyll作为博客开发框架

我之前粗略地比较了下hexo和jekyll，应该说两者使用起来都比较简单。不过jekyll有如下优势：

1. github page官方使用jekyll。

   我觉得选择一个框架应该注重它的发展前途，目前jekyll用户规模以及维护的活跃度都比hexo高。总体来说会更加稳定（bug修复得更快？

2. jekyll的使用框架简单明了。

   基本上花一上午读一遍官方文档，你就能对其原理有个大致理解。之后出了什么问题你也可以尝试自己解决，而不必各种google。

## 一个博客系统应该具备哪些特点？

1. 页面应该简单明了。

   现在的博客主题五花八门，你可以自己选一个中意的主题进行部署。不过我个人的喜好是简单直接点的，太过花哨容易分散我的注意力，将精力集中在博客内容的撰写上才是正途。

2. 开发部署应该尽可能简单。

   每次写篇markdown都要敲一堆命令才能渲染部署未免太过麻烦，可以考虑引入`makefile`来进行管理。

3. 需要支持定制化。

   毕竟博客系统最主要的功能是帮助自己进行文档的整理、记录与查询，每个人可能都有自己不同的需求，所以自己配置自己的主题与框架可能更合适些。

   我的博客目前支持的功能如下：

   * 文件搜索功能

     当撰写的博客达到一定规模时，手动检索就太慢了，因此加了一个关键字搜索框。

   * 归类功能

     emmmm，应该算是最最最基本的功能？总得让文章组织布局看起来有点逻辑吧。
