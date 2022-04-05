一般我们要搭建一个博客网站，需要：
1. 租赁一个服务器
2. 搭建一个博客应用
3. 编写博客内容
4. 部署博客应用
5. 服务器维护
6. ...

很多时候，我们只想单纯地写个博客。并不想去维护除了博客内容外的其他东西，什么服务器、什么博客应用等等。有没有可能把这些事情找个“人”来做，并且免费呢？
### github-pages

> `github-pages`是`github`面向用户、组织和项目开放的公共静态页面搭建托管服务。

这不就是我们想要的吗？而且`github-pages`稳定，无限流量。只需要把博客托管到`github-pages`上即可，无需关心博客应用、主机等其他事情了。
`github-pages`只能托管静态的站点，那么怎么来生成静态站点呢？

### hexo

> Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用
> Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

下面具体看看怎么来利用hexo + github-pages来搭建静态博客（mac环境，其他环境略有不同）：

### 安装hexo
1. 首先安装nodejs: `brew install node`
2. 安装[git](https://git-scm.com/)
3. 安装hexo: `npm install -g hexo-cli`

### 建站
1. 初始化hexo目录：`hexo init <folder>`
2. 然后进入第一步中的目录，执行`npm install`
3. 配置`_config.yml`, 网站的大部分配置都维护在这个配置文件中。另外主题也会维护一些和主题相关的配置。

### 部署
1. 安装git部署模块
    在本地博客目录中执行：`npm install hexo-deployer-git --save`
2. 创建github-pages
    在github上创建一个项目，例如blog（如无账号请，先申请账号）
![](/images/build-blog-using-hexo-and-github-pages-1.png)
3. 将初始化的博客传到git上（这里我们上传到master分支上，地址则为第五步项目地址）
   在`_config.xml`中配置：
    ```
    deploy:
  type: git
  repo: https://github.com/gavincook/blog.git
  branch: master
    ```
    然后在博客本地目录执行：`hexo g -d`

4. 在github项目设置中打开github-pages，并指定分支
![](/images/build-blog-using-hexo-and-github-pages-2.png)
这样就完成了静态博客生成，并同步到github上。此时使用github-pages提供的url即可访问。如需配置自定义域名，参见

### 写作
1. 执行`hexo new <title>`创建一个新的文档，如果`title`如果包含空格，请用引号将`title`引起来。
2. 此时会在`source/_posts`中生成对应的文档（markdown文档）。编辑此markdown文档即可。
3. 在本地编写博客时，可以通过`hexo server`启动本地服务，然后通过`localhost:4000`（默认）进行访问。
4. 写作完成后，执行一次`hexo g -d`即可同步

### 插件安装

1. 友言评论插件
首先在友言上注册账号，然后将代码复制到`themes/主题/partial/comments.ejs`或其包含在body里的模板。具体视主题的文件结构为准。
    ```
    <% if(page.comments && theme.youyan.on) { %>
    <!-- UY BEGIN -->
    <div id="uyan_frame"></div>
    <script type="text/javascript" src="http://v2.uyan.cc/code/uyan.js?uid=申请的id"></script>
    <!-- UY END -->
    <% } %>
    ```

2. 百度统计插件
在`themes/主题/partial/footer.ejs`或其他通用的模板里引入：
    ```
    <script>
    var _hmt = _hmt || [];
    (function() {
      var hm = document.createElement("script");
      hm.src = "https://hm.baidu.com/hm.js?申请的标识";
      var s = document.getElementsByTagName("script")[0]; 
      s.parentNode.insertBefore(hm, s);
    })();
    </script>
    ```
3. jiathis分享插件
在和统计插件同样的地方加入：

    ```
    <!-- JiaThis Button BEGIN -->
    <script type="text/javascript">
    var jiathis_config = {data_track_clickback:'true'};
    </script>
    <script type="text/javascript" src="http://v3.jiathis.com/code/jiathis_r.js?type=left&amp;move=0&amp;uid=申请的id" charset="utf-8"></script>
    <!-- JiaThis Button END -->
    ```

### 自定义域名映射
1. 增加域名解析CNAME，将地址指向`github-pages`的域名即可。比如`gavincook.github.io`
2. 然后在本地博客的`source/`目录中增加一个名为`CNAME`的文件，内容就为需要配置的域名
3. 然后将本地博客部署到github即可，这时就能通过自定义域名访问博客了

