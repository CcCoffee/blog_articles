本文介绍如何让个人网站使用个性的艺术字类型的 favicon.ico。
## 生成 favicon.ico 图片
1. 生成艺术字图片并下载

   http://www.akuziti.com/

2. 上传图片截选并生成ico格式文件

   https://tool.lu/favicon/

## 使用 favicon.ico
1. 将成功生成的图标文件下载并改名为 favicon.ico，上传到网站任意目录。 
2. 在网站首页的源文件 head 之间插入下面的代码，href中写上 favicon.ico 的相对地址:

   ```javascript
   <link rel="shortcut icon" href="/favicon.ico"/>
   <link rel="bookmark" href="/favicon.ico"/>
   ```
