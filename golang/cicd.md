# CI CD

## git lfs

```
# 安装完后，需要运行以下命令，每台机器只需运行一次, 运行多次也没有问题
$ git lfs install
 
 
# 将 LFS服务器地址配置为 大文件 服务器
$ git config --global lfs.url "xxx"
# 确认环境
git lfs env

# 大文件标记
git lfs track "*.a"
git lfs track "*.h264"

git lfs untrack "*.a

git lfs pull

## 提交

# 确保 .gitattributes 被git追踪
git add .gitattributes
git add video.h264 common.a
git commit "add static library and video"
```
