# Blogs

博客源代码

## 创建博客相关命令

1. 新建博客 `hexo n 'xxx'`
2. 启动本地服务器 `hexo s`
3. 生成网站静态文件到默认设置的 public 文件夹 `hexo g`
4. 自动生成网站静态文件，并部署到设定的仓库 `hexo d`

## 主题

1. 安装 next 主题 `git clone https://github.com/theme-next/hexo-theme-next.git themes/next`

2. 修改主题

  在 `themes/next/_config.yml` 文件中修改对应配置

  ```yml
  favicon:
    small: /images/favicon-16x16-next.png
    medium: /images/favicon-32x32-next.png
  
  rss: /atom.xml

  footer:
    icon:
      animated: false
    powered:
      enable: true
    theme:
      enable: false

  menu:
    home: / || home
    archives: /archives/ || archive
    tags: /tags/ || tags
    categories: /categories/ || th
    about: /about/ || user

  scheme: Gemini

  social:
    GitHub: https://github.com/RaulZuo || github

  avatar:
    url: /images/avatar.png

  auto_excerpt:
    enable: true

  highlight_theme: night eighties
  ```