# 关于Hexo电脑迁移备份的问题

## 首先我们要做blog文件夹的备份
1. 先将我们的blog文件夹上传至github，新建一个存储仓库名字就叫HexoBack

2. 将blog文件目录作为存储仓库推送到github远端仓库上

## 接下来我们要模拟在另一台电脑下载blog工程

1. 这里我使用的是Source Tree这个软件从github上clone下来了blog工程
2. 我们打开命令行工具，这里我用的是git的窗口，输入命令 
    ```
    hexo clean
    ```
    但是并没有成功运行，因为新电脑并没有一些基本的配置工具

3. 我们需要下载一些基本的环境 
   - nodejs

        我们需要到nodejs官网下载，安装步骤很简单这里就不说了，一直下一步就行了，他附带这npm包管理器
   - cnpm 因为国内npm下载很慢 所以我们利用npm来下载淘宝的镜像源cnpm

        下载好nodejs后输入命令
    ```
    npm install -g cnpm --registry=https://registry.npm.taobao.org
    ```
4. 下载Hexo博客框架
    ```
    cnpm install -g hexo-cli
    ```
5. 然后我们就可以安装hexo命令了输入命令
    ```
    cnpm install hexo --save
    ```
6. 输入命令检查版本信息
    ```
    hexo -v
    ```
7. 以上步骤正确的话我们就可以开始使用hexo写博客了

    下面分别是清理和生成博客文件的命令
    ```
    hexo clean
    hexo g
    ```
8. 当我们要部署到远端的时候，输入命令
    ```
    hexo d
    ```
    发现提示我们 
    >ERROR Deployer not found:git

    这是因为我们需要再安装一个部署到远端的插件，所以我们继续使用cnpm来下载这个插件
    ```
    cnpm install --save hexo-deployer-git
    ```
    下载好了我们就可以执行刚才的命令部署到远端了
## 注意事项
1. 首先每次换电脑之前确保要将本地的blog工程最新版推送至github上
2. 每次更换电脑后首先要将最新版的blog目录从github上 pull 到本地仓库

## 这样我们就完美解决了更换电脑的问题了