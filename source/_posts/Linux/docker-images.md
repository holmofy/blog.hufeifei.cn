---
title: 解决Docker镜像爆满的问题
date: 2022-02-13
categories: Linux运维
tags: Linux
---

## 现象 
使用过docker的人都知道，在正常情况下。我们使用multi-stage构建利用docker镜像缓存机制，可以加快构建速度。

但是缓存的镜像一多，没有及时释放磁盘空间，磁盘就容易爆满。

```diff
[root@report ~]# df -h
文件系统        容量  已用  可用 已用% 挂载点
devtmpfs        909M     0  909M    0% /dev
tmpfs           919M   24K  919M    1% /dev/shm
tmpfs           919M  752K  919M    1% /run
tmpfs           919M     0  919M    0% /sys/fs/cgroup
/dev/vda1        50G  9.6G   38G   21% /
tmpfs           184M     0  184M    0% /run/user/0
+overlay          50G  9.6G   38G   21% /var/lib/docker/overlay2/eefc0488163e0d222c384c54041549023a998e3fefeea1c672f0cb9b0f1e78a1/merged
shm              64M     0   64M    0% /var/lib/docker/containers/ceb6efe9a7cd0563af63ee0b37fb7d7d4f387a46691e7d1d6109876a22fc8b6d/mounts/shm
+overlay          50G  9.6G   38G   21% /var/lib/docker/overlay2/038df81678b0660a7f3e9eaade223fdd541beff0799bed64437eabe525c4088d/merged
+overlay          50G  9.6G   38G   21% /var/lib/docker/overlay2/1bd852511b0179c8cc31b9e5a3b6ccf66da6d990a98a1b5ca7874e556f1e384e/merged
shm              64M     0   64M    0% /var/lib/docker/containers/83e5b18fe998f7ed3ceb4b6328ee427748cd8fd8f1de313222286e3bc125bb15/mounts/shm
[root@report ~]# docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          12        8         3.453GB   903.7MB (26%)
Containers      9         3         120.3MB   13.79MB (11%)
Local Volumes   1         1         292.6MB   0B (0%)
Build Cache     0         0         0B        0B
```

如果每次构建完手动`docker rmi`又达不到加快构建速度的效果。

尤其是在持续集成环境中，大家公用一个build machine的时候。大家各自打扫门前雪，更加不会有人care磁盘会不会被占满。 

## 方法 

为了一劳永逸的解决这个问题，最好的办法莫过于通过定时任务来清理旧的image。 这个方法听起来高大上，用起来简单的很。 运行crontab -e命令编辑定时任务。 

```bash
crontab -e
```

在打开的文本编辑器最后添加如下一行，然后保存退出。

```bash
0 1 * * * docker image prune -a --force --filter "until=48h"
```

然后执行下面的命令使定时任务生效。

```bash
systemctl restart crond.service
```

其实，到这里，整个配置就结束了。接下来我们简单解释一下。 
上面的定时任务是每天夜里1点钟删除2天（48h）之前的image。
具体的操作时间，具体的image保留时间，可以根据自己的情况修改。
