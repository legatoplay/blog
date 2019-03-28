---
title: Hexo 简单使用
subtitle: 111
date: 2019-03-28 14:35:10
tags: [Hexo,JS]
category: 前端
---
1. npm安装

   ```bash
   npm install hexo -g
   ```

2. 查看版本号

   ```bash
   hexo -v
   ```
 <!-- more -->

3. 初始化项目

   ```bash
   hexo init
   ```

4. 发布

   ```bash
   hexo g
   ```

5. 启动

   ```bash
   hexo server -p 8080
   ```

6. 创建新页面

   ```bash
   hexo new post "***"
   ```

7. 发布到github

   安装github发布扩展

   ```bash
   npm install hexo-deployer-git --save
   ```

   配置__config.yml，末尾添加

   ```yaml
   # Deployment
   ## Docs: https://hexo.io/docs/deployment.html
   deploy:
     type: git
     repository: git@github.com:legatoplay/legatoplay.github.io.git
     branch: master
   ```

   发布到github

   ```bash
   hexo d -g
   ```
