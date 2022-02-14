# runc
基本使用
```
# create the top most bundle directory
mkdir /mycontainer
cd /mycontainer

# create the rootfs directory
mkdir rootfs

# export busybox via Docker into the rootfs directory
docker export $(docker create busybox) | tar -C rootfs -xvf -

runc spec
修改如下参数：
"terminal": false and "args": ["sleep", "5"]

runc create mycontainerid

# view the container is created and in the "created" state
runc list

# start the process inside the container
runc start mycontainerid

# after 5 seconds view that the container has exited and is now in the stopped state
runc list

# now delete the container
runc delete mycontainerid
```
## 关键函数
- startContainer
- createContainer 

## libcontainer
- New
- LinuxFactory ：实现Factory
- linuxContainer：实现Container
- Factory ：接口
- Container ： 容器
