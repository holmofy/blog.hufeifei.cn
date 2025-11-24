---
title: inode占满的问题
date: 2025-04-08
categories: Linux运维
tags: Linux
---

今天又遇到一个神奇的问题，磁盘呢命名才65%，却死活无法创建文件。`df -h`查看磁盘占用完全正常：

```sh
root@gateway:~# df -h
文件系统        大小  已用  可用 已用% 挂载点
udev            961M     0  961M    0% /dev
tmpfs           197M  1.9M  195M    1% /run
/dev/vda1        40G   26G   14G   65% /
tmpfs           984M   24K  984M    1% /dev/shm
tmpfs           5.0M     0  5.0M    0% /run/lock
tmpfs           197M     0  197M    0% /run/user/0
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/63203e0c37460d7a69334b0668eae9aaae178982aaa3c469aa3e907fc7f898f0/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/2a520537cee59aafb67a8114d533755ef51848ce40d48baace86b7b462738a23/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/3b65d999278fa90e41008d4f411eadec6eefa83c75f621060663fe8e11a475b6/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/2fe9d6e5296d6d45f65e4276533e87260c63ce6e1bab657156ea6941c1f1c74d/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/8881672477f731dd0f0dbf75645a0bb26a4f479352aa1b195627526eb4e944e5/merged
tmpfs           984M  4.0K  984M    1% /var/lib/docker/containers/c264e6398fed6066e3df74e2791ce0a382812602d216c5de342c1a5fb4c03ae5/mounts/secrets
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/9eac45b64505face3650ec66f380eeed4e4f09c013192c0fe1e971f2e1e87271/merged
tmpfs           984M  4.0K  984M    1% /var/lib/docker/containers/f823d2fdf4aafb87baa214547e7b992c5983492b19a0b22254a074ebd9afb9f3/mounts/secrets
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/59c16c23f0656b48c249fa378fd9108f7b393a8b4a6504f6a6fc1eaa8ce297dd/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/9117355cb7c22323fe4ac4fc97afebe6365f2c9259e5fa7c13d2ebb626720d9c/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/b6a567a82e1be55b820e9951cf2f6ea712de163a3f24ed05ff65ee981b842c81/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/76f0fbf65b59f955d396489e34966e38478df451c00c2def7a2b66699c6ddc8b/merged
overlay          40G   26G   14G   65% /var/lib/docker/overlay2/b446aa32f08ddef9fc80959f39f6a81d4584eeb3b51f132bae58de93a1aca34b/merged
tmpfs           984M   12K  984M    1% /var/lib/docker/containers/499b037d601751016c4e9e0086dce5704a30bcd58a44797f5fd1f314e537fa64/mounts/secrets
```

启动容器也死活起不来，按照GPT的建议`df -i`看了一下，原来是inode耗尽了：

```sh
root@gateway:~# df -i
文件系统        Inodes   已用I  可用I 已用I% 挂载点
udev            245913     343 245570     1% /dev
tmpfs           251772     884 250888     1% /run
/dev/vda1      2621440 2620957    483   100% /
tmpfs           251772       7 251765     1% /dev/shm
tmpfs           251772       3 251769     1% /run/lock
tmpfs            50354      21  50333     1% /run/user/0
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/63203e0c37460d7a69334b0668eae9aaae178982aaa3c469aa3e907fc7f898f0/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/2a520537cee59aafb67a8114d533755ef51848ce40d48baace86b7b462738a23/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/3b65d999278fa90e41008d4f411eadec6eefa83c75f621060663fe8e11a475b6/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/2fe9d6e5296d6d45f65e4276533e87260c63ce6e1bab657156ea6941c1f1c74d/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/8881672477f731dd0f0dbf75645a0bb26a4f479352aa1b195627526eb4e944e5/merged
tmpfs           251772       2 251770     1% /var/lib/docker/containers/c264e6398fed6066e3df74e2791ce0a382812602d216c5de342c1a5fb4c03ae5/mounts/secrets
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/9eac45b64505face3650ec66f380eeed4e4f09c013192c0fe1e971f2e1e87271/merged
tmpfs           251772       2 251770     1% /var/lib/docker/containers/f823d2fdf4aafb87baa214547e7b992c5983492b19a0b22254a074ebd9afb9f3/mounts/secrets
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/59c16c23f0656b48c249fa378fd9108f7b393a8b4a6504f6a6fc1eaa8ce297dd/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/9117355cb7c22323fe4ac4fc97afebe6365f2c9259e5fa7c13d2ebb626720d9c/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/b6a567a82e1be55b820e9951cf2f6ea712de163a3f24ed05ff65ee981b842c81/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/76f0fbf65b59f955d396489e34966e38478df451c00c2def7a2b66699c6ddc8b/merged
overlay        2621440 2620957    483   100% /var/lib/docker/overlay2/1b29e0402874d514ee14fb82a2f965255dcfd1cba81d3ec78da1ff3f931c9b1e/merged
tmpfs           251772       3 251769     1% /var/lib/docker/containers/255be73d1a8a7d9d14c49a6c29e900a7717ef9518e6c90c1de054e309d2548f9/mounts/secrets
```

一般出现这种情况就是创建了大量的小文件导致linux操作系统耗尽了。

深入检查后发现是arroyo创建了大量的checkpoint文件

```sh
root@gateway:/var/lib/docker/overlay2/123c7bb87ead4c7076c138d241409cc91fb603b48b90ba60bd765e082c799f37/diff/tmp/arroyo/checkpoints# ls job_6Apv6plkC9/checkpoints/
checkpoint-0000001  checkpoint-0000012	checkpoint-0000023  checkpoint-0000034	checkpoint-0000045  checkpoint-0000056	checkpoint-0000067
checkpoint-0000002  checkpoint-0000013	checkpoint-0000024  checkpoint-0000035	checkpoint-0000046  checkpoint-0000057	checkpoint-0000068
checkpoint-0000003  checkpoint-0000014	checkpoint-0000025  checkpoint-0000036	checkpoint-0000047  checkpoint-0000058	checkpoint-0000069
checkpoint-0000004  checkpoint-0000015	checkpoint-0000026  checkpoint-0000037	checkpoint-0000048  checkpoint-0000059	checkpoint-0000070
checkpoint-0000005  checkpoint-0000016	checkpoint-0000027  checkpoint-0000038	checkpoint-0000049  checkpoint-0000060	checkpoint-0000071
checkpoint-0000006  checkpoint-0000017	checkpoint-0000028  checkpoint-0000039	checkpoint-0000050  checkpoint-0000061	checkpoint-0000072
checkpoint-0000007  checkpoint-0000018	checkpoint-0000029  checkpoint-0000040	checkpoint-0000051  checkpoint-0000062	checkpoint-0000073
checkpoint-0000008  checkpoint-0000019	checkpoint-0000030  checkpoint-0000041	checkpoint-0000052  checkpoint-0000063	checkpoint-0000074
checkpoint-0000009  checkpoint-0000020	checkpoint-0000031  checkpoint-0000042	checkpoint-0000053  checkpoint-0000064	checkpoint-0000075
checkpoint-0000010  checkpoint-0000021	checkpoint-0000032  checkpoint-0000043	checkpoint-0000054  checkpoint-0000065	checkpoint-0000076
checkpoint-0000011  checkpoint-0000022	checkpoint-0000033  checkpoint-0000044	checkpoint-0000055  checkpoint-0000066
root@gateway:/var/lib/docker/overlay2/123c7bb87ead4c7076c138d241409cc91fb603b48b90ba60bd765e082c799f37/diff/tmp/arroyo/checkpoints# ls job_I2aZfI4evf/checkpoints/
checkpoint-0000001  checkpoint-0005481	checkpoint-0010961  checkpoint-0016441	checkpoint-0021921  checkpoint-0027401	checkpoint-0032881
checkpoint-0000002  checkpoint-0005482	checkpoint-0010962  checkpoint-0016442	checkpoint-0021922  checkpoint-0027402	checkpoint-0032882
checkpoint-0000003  checkpoint-0005483	checkpoint-0010963  checkpoint-0016443	checkpoint-0021923  checkpoint-0027403	checkpoint-0032883
checkpoint-0000004  checkpoint-0005484	checkpoint-0010964  checkpoint-0016444	checkpoint-0021924  checkpoint-0027404	checkpoint-0032884
checkpoint-0000005  checkpoint-0005485	checkpoint-0010965  checkpoint-0016445	checkpoint-0021925  checkpoint-0027405	checkpoint-0032885
checkpoint-0000006  checkpoint-0005486	checkpoint-0010966  checkpoint-0016446	checkpoint-0021926  checkpoint-0027406	checkpoint-0032886
checkpoint-0000007  checkpoint-0005487	checkpoint-0010967  checkpoint-0016447	checkpoint-0021927  checkpoint-0027407	checkpoint-0032887
checkpoint-0000008  checkpoint-0005488	checkpoint-0010968  checkpoint-0016448	checkpoint-0021928  checkpoint-0027408	checkpoint-0032888
checkpoint-0000009  checkpoint-0005489	checkpoint-0010969  checkpoint-0016449	checkpoint-0021929  checkpoint-0027409	checkpoint-0032889
checkpoint-0000010  checkpoint-0005490	checkpoint-0010970  checkpoint-0016450	checkpoint-0021930  checkpoint-0027410	checkpoint-0032890
checkpoint-0000011  checkpoint-0005491	checkpoint-0010971  checkpoint-0016451	checkpoint-0021931  checkpoint-0027411	checkpoint-0032891
checkpoint-0000012  checkpoint-0005492	checkpoint-0010972  checkpoint-0016452	checkpoint-0021932  checkpoint-0027412	checkpoint-0032892
checkpoint-0000013  checkpoint-0005493	checkpoint-0010973  checkpoint-0016453	checkpoint-0021933  checkpoint-0027413	checkpoint-0032893
checkpoint-0000014  checkpoint-0005494	checkpoint-0010974  checkpoint-0016454	checkpoint-0021934  checkpoint-0027414	checkpoint-0032894
checkpoint-0000015  checkpoint-0005495	checkpoint-0010975  checkpoint-0016455	checkpoint-0021935  checkpoint-0027415	checkpoint-0032895
checkpoint-0000016  checkpoint-0005496	checkpoint-0010976  checkpoint-0016456	checkpoint-0021936  checkpoint-0027416	checkpoint-0032896
checkpoint-0000017  checkpoint-0005497	checkpoint-0010977  checkpoint-0016457	checkpoint-0021937  checkpoint-0027417	checkpoint-0032897
checkpoint-0000018  checkpoint-0005498	checkpoint-0010978  checkpoint-0016458	checkpoint-0021938  checkpoint-0027418	checkpoint-0032898
```

把这个docker的overlay2文件夹删掉，立马清爽了：

```sh
root@gateway:~# df -i
文件系统        Inodes  已用I   可用I 已用I% 挂载点
udev            245913    343  245570     1% /dev
tmpfs           251772    884  250888     1% /run
/dev/vda1      2621440 227508 2393932     9% /
tmpfs           251772      7  251765     1% /dev/shm
tmpfs           251772      3  251769     1% /run/lock
tmpfs            50354     21   50333     1% /run/user/0
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/63203e0c37460d7a69334b0668eae9aaae178982aaa3c469aa3e907fc7f898f0/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/2a520537cee59aafb67a8114d533755ef51848ce40d48baace86b7b462738a23/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/3b65d999278fa90e41008d4f411eadec6eefa83c75f621060663fe8e11a475b6/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/2fe9d6e5296d6d45f65e4276533e87260c63ce6e1bab657156ea6941c1f1c74d/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/8881672477f731dd0f0dbf75645a0bb26a4f479352aa1b195627526eb4e944e5/merged
tmpfs           251772      2  251770     1% /var/lib/docker/containers/c264e6398fed6066e3df74e2791ce0a382812602d216c5de342c1a5fb4c03ae5/mounts/secrets
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/9eac45b64505face3650ec66f380eeed4e4f09c013192c0fe1e971f2e1e87271/merged
tmpfs           251772      2  251770     1% /var/lib/docker/containers/f823d2fdf4aafb87baa214547e7b992c5983492b19a0b22254a074ebd9afb9f3/mounts/secrets
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/59c16c23f0656b48c249fa378fd9108f7b393a8b4a6504f6a6fc1eaa8ce297dd/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/9117355cb7c22323fe4ac4fc97afebe6365f2c9259e5fa7c13d2ebb626720d9c/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/b6a567a82e1be55b820e9951cf2f6ea712de163a3f24ed05ff65ee981b842c81/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/76f0fbf65b59f955d396489e34966e38478df451c00c2def7a2b66699c6ddc8b/merged
overlay        2621440 227508 2393932     9% /var/lib/docker/overlay2/1b29e0402874d514ee14fb82a2f965255dcfd1cba81d3ec78da1ff3f931c9b1e/merged
tmpfs           251772      3  251769     1% /var/lib/docker/containers/255be73d1a8a7d9d14c49a6c29e900a7717ef9518e6c90c1de054e309d2548f9/mounts/secrets
```

查了一下[arroyo的相关文档](https://github.com/ArroyoSystems/arroyo/blob/ccfb58f5/crates/arroyo-rpc/default.toml#L15)，需要把checkpoint的compact功能开启

```toml
[pipeline.compaction]  
enabled = true  
checkpoints-to-compact = 4
```

docker启动的话可以通过环境变量进行设置

```sh
- ARROYO__PIPELINE__COMPACTION__ENABLED=true  
- ARROYO__PIPELINE__COMPACTION__CHECKPOINTS_TO_COMPACT=4
```

然后还要设置一下`CHECKPOINTS_TO_KEEP`来确定arroyo要保留多少个checkpoint文件。
