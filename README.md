# [个人博客](https://bufferflies.github.io/)
## 本地运行
hugo server

## 上传
hugo -d ./docs  生成静态文件到docs目录
git add .
git commit -m ""
git push origin main 

## 创建文章

hugo new file.md

eg:
hugo new post/first.md


## 添加脚本
在 ./theme/jane/layouts/scripts.html 进行

## 更换代码样式
1. 点击[这里](https://xyproto.github.io/splash/docs/)选择代码样式
2. 执行如下命令更新样式  这里使用monokai

```
hugo gen chromastyles --style=monokai > syntax.css
```

3. 修改config.toml