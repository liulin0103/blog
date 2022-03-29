---
title: 平滑下线 Kudu Tablet Server
date: 2022/03/21 14:42:00
updated: 2022/03/21 14:42:00
categories:
- Kudu
tags:
- Kudu
---

## 平滑下线 Kudu Tablet Server

1. 在 Cloudera Manager 界面上停止该节点的 Tablet Server 角色服务

   ![](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/20220321144555.png)

   <!--more-->

2. 编写脚本 `KuduRemoveTabletServer.sh` 

   ```bash
   #! /bin/bash
   
   # 使用方法：
   # sh KuduRemoveTabletServer.sh <集群地址> <需要迁移的TabletServerUUID> <集群中任一存活的一个服务节点>
   
   echo "input kudu master:$1 , taletId:$2, tabletName:$3 , go on? yes/no"
   
   read value
   
   if [ $value == yes ]; then
       kududata=$(kudu remote_replica list $3 | grep 'Tablet id')
       IFS=$'\n'
       for tablet in $kududata; do
           echo '\n------------------------------------------------------------------------------------------'
           tabletid=${tablet#Tablet id: }
           echo "sudo -u kudu kudu tablet change_config remove_replica $1 $tabletid $2"
           sudo -u kudu kudu tablet change_config remove_replica $1 $tabletid $2
       done
   fi
   ```

   > 脚本的作用是从另外一个存活的 ts 节点读取所有的 TabletId，遍历这些 id 并从要下线的 ts 中删除这些 tablet。

3. 从 Kudu 集群中删除要下线节点的所有 tablet 数据， kudu 集群会自动选取其他节点替换该节点的服务。

4. 保证余下的每一个存活 ts 节点的 tablet 都没有副本存在于要下线的 ts 节点。如果某个 ts 中还有 tablet 的副本存在于要下线的 ts 中，可以使用这个 ts 执行一次 `KuduRemoveTabletServer.sh` 脚本。

   ![image-20220321145637109](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220321145637109.png)

5. 等待kudu集群自动恢复，可以使用ksck命令检查集群健康状态是否正常。

   ```bash
   kudu cluster ksck master1:7051,master2:7051,master3:7051
   ```

6. 保证所有副本全部可用的情况下。可以在 CM 中删除下线节点的 ts 角色。

   ![image-20220321150814478](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220321150814478.png)

7. 删除后 `ksck` 依旧会显示下线的节点处于 dead 状态。

   ![image-20220321150907068](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220321150907068.png)

8. 滚动重启 kudu master server 即可清除已经下线的 tablet server。

   

   

