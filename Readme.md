# HEXO 博客配置说明
## 开发环境
1. 安装node
2. 使用node安装hexo 
```shell
npm install -g hexo-cli
```
3. 安装主题
- 在https://hexo.io/themes/找到心仪的主题
- 解压主题到 /thmes/ 路径下，并修改路径名称为主题名称
- 在./_config.yml 设置启用当前的主题
- 根据主题的需要安装node依赖，本博客使用的Apollo主题，需要安装
```shell
npm install --save hexo-renderer-jade hexo-generator-feed hexo-generator-sitemap hexo-browsersync hexo-generator-archive
```

## 与github pages集成
1. 新建github仓库 c101088.github.io
2. 安装hexo部署插件
```shell
npm install hexo-deployer-git --save
```
3. 修改配置文件./_config.yml
```yml
deploy:
  type: git
  repository: git@github.com:c101088/c101088.github.io.git
  branch: main
```
4. 生成ssh key，让本地可以直接访问github
- 本地git-bash ,生成pubkey
- 将pubkey保存到github设置

## 命令记录
```shell
- hexo s  #启动本地预览
- hexo d  # 将本地部署到远程github
```
