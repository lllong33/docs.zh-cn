# 手动部署 StarRocks

本文介绍如何手动部署 StarRocks。其他安装方式请参考[部署概览](../deployment/deployment_overview.md)。

## 第一步：启动 FE 服务

以下操作在 FE 实例上执行。

1. 创建元数据存储路径。建议将元数据存储在与 FE 部署文件不同的路径中。请确保此路径存在并且您拥有写入权限。

   ```Bash
   # 将 <meta_dir> 替换为您要创建的元数据目录。
   mkdir -p <meta_dir>
   ```

2. 进入先前准备好的 [StarRocks FE 部署文件](../deployment/prepare_deployment_files.md)所在路径，修改 FE 配置文件 **fe/conf/fe.conf**。

   a. 在配置项 `meta_dir` 中指定元数据路径。

      ```YAML
      # 将 <meta_dir> 替换为您已创建的元数据目录。
      meta_dir = <meta_dir>
      ```

   b. 如果任何在 [环境配置清单](../deployment/environment_configurations.md) 中提到的 FE 端口被占用，您必须在 FE 配置文件中为其分配其他可用端口。

      ```YAML
      http_port = aaaa        # 默认值：8030
      rpc_port = bbbb         # 默认值：9020
      query_port = cccc       # 默认值：9030
      edit_log_port = dddd    # 默认值：9010
      ```

      > **注意**
      >
      > 如需在集群中部署多个 FE 节点，您必须为所有 FE 节点分配相同的 `http_port`。

   c. 如需为集群启用 IP 地址访问，您必须在配置文件中添加配置项 `priority_networks`，为 FE 节点分配一个专有的 IP 地址（CIDR格式）。如需为集群启用 FQDN 访问，则可以忽略该配置项。

      ```YAML
      priority_networks = x.x.x.x/x
      ```

      > **说明**
      >
      > 您可以在终端中运行 `ifconfig` 以查看当前实例拥有的 IP 地址。

   d. 如果您的实例安装了多个 JDK，并且您要使用 JDK 与环境变量 `JAVA_HOME` 中指定的不同，则必须在配置文件中添加配置项 `JAVA_HOME` 来指定所选该 JDK 的安装路径。

      ```YAML
      # 将 <path_to_JDK> 替换为所选 JDK 的安装路径。
      JAVA_HOME = <path_to_JDK>
      ```

   e.  更多高级配置项请参考 [参数配置 - FE 配置项](../administration/Configuration.md#fe-配置项)。

3. 启动 FE 节点。

   - 如需为集群启用 IP 地址访问，请运行以下命令启动 FE 节点：

     ```Bash
     ./fe/bin/start_fe.sh --daemon
     ```

   - 如需为集群启用 FQDN 访问，请运行以下命令启动 FE 节点：:

     ```Bash
     ./fe/bin/start_fe.sh --host_type FQDN --daemon
     ```

     您只需在第一次启动节点时指定参数 `--host_type`。

     > **注意**
     >
     > 如需启用 FQDN 访问，在启动 FE 节点之前，请确保您已经在 **/etc/hosts** 中为所有实例分配了主机名。有关详细信息，请参考 [环境配置清单 - 主机名](../deployment/environment_configurations.md#主机名)。

4. 查看 FE 日志，检查 FE 节点是否启动成功。

   ```Bash
   cat fe/log/fe.log | grep thrift
   ```

   如果日志打印以下内容，则说明该 FE 节点启动成功：

   "2022-08-10 16:12:29,911 INFO (UNKNOWN x.x.x.x_9010_1660119137253(-1)|1) [FeServer.start():52] thrift server started with port 9020."

## 第二步：（可选）部署高可用 FE 集群

高可用的 FE 集群需要在 StarRocks 集群中部署至少三个 Follower FE 节点。如需部署高可用的 FE 集群，您需要额外再启动两个新的 FE 节点。

**除用于启动 FE 节点的命令外**，您可以重复第一步中的过程启动新的 FE 节点。当向集群中添加新的 FE 节点时，您必须为新的 FE 节点分配一个 helper 节点（本质上是一个现有的 FE 节点）以同步所有 FE 节点的元数据信息。

- 如已为集群启用 IP 地址访问，请运行以下命令启动 FE 节点：

  ```Bash
  # Replace <helper_fe_ip> with the IP address (priority_networks) 
  # of the helper node, and replace <helper_edit_log_port> with 
  # the helper node's edit_log_port.
  ./fe/bin/start_fe.sh --helper <helper_fe_ip>:<helper_edit_log_port> --daemon
  ```

  您只需在第一次启动节点时指定参数 `--helper`。

- 如已为集群启用 FQDN 访问，请运行以下命令启动 FE 节点：

  ```Bash
  # Replace <helper_fqdn> with the FQDN of the helper node, 
  # and replace <helper_edit_log_port> with the helper node's edit_log_port.
  ./fe/bin/start_fe.sh --helper <helper_fqdn>:<helper_edit_log_port> \
      --host_type FQDN --daemon
  ```

  您只需在第一次启动节点时指定参数 `--helper` 和 `--host_type`。

## 第三步：启动 BE 服务

以下操作在 BE 实例上执行。

1. 创建数据存储路径。建议将数据存储在与 BE 部署文件不同的路径中。请确保此路径存在并且您拥有写入权限。

   ```Bash
   # 将 <storage_root_path> 替换为您要创建的数据存储路径。
   mkdir -p <storage_root_path>
   ```

2. 进入先前准备好的 [StarRocks BE 部署文件](../deployment/prepare_deployment_files.md)所在路径，修改 BE 配置文件 **be/conf/be.conf**。

   a. 在配置项 `storage_root_path` 中指定数据存储路径。

      ```YAML
      # 将 <storage_root_path> 替换为您创建的数据存储路径。
      storage_root_path = <storage_root_path>
      ```

   b. 如果任何在 [环境配置清单](../deployment/environment_configurations.md) 中提到的 BE 端口被占用，您必须在 BE 配置文件中为其分配其他可用端口。

      ```YAML
      be_port = vvvv                   # 默认值：9060
      be_http_port = xxxx              # 默认值：8040
      heartbeat_service_port = yyyy    # 默认值：9050
      brpc_port = zzzz                 # 默认值：8060
      ```

   c. 如需为集群启用 IP 地址访问，您必须在配置文件中添加配置项 `priority_networks`，为 BE 节点分配一个专有的 IP 地址（CIDR格式）。如需为集群启用 FQDN 访问，则可以忽略该配置项。

      ```YAML
      priority_networks = x.x.x.x/x
      ```

      > **说明**
      >
      > 您可以在终端中运行 `ifconfig` 以查看当前实例拥有的 IP 地址。

   d. 如果您的实例安装了多个 JDK，并且您要使用 JDK 与环境变量 `JAVA_HOME` 中指定的不同，则必须在配置文件中添加配置项 `JAVA_HOME` 来指定所选该 JDK 的安装路径。

      ```YAML
      # 将 <path_to_JDK> 替换为所选 JDK 的安装路径。
      JAVA_HOME = <path_to_JDK>
      ```

   e.  更多高级配置项请参考 [参数配置 - BE 配置项](../administration/Configuration.md#be-配置项)。

3. 启动 BE 节点。

   ```Bash
   ./be/bin/start_be.sh --daemon
   ```

   > **注意**
   >
   > - 如需启用 FQDN 访问，在启动 BE 节点之前，请确保您已经在 **/etc/hosts** 中为所有实例分配了主机名。有关详细信息，请参考 [环境配置清单 - 主机名](../deployment/environment_configurations.md#主机名)。
   > - 启动 BE 节点时无需指定参数 `--host_type`。

4. 查看 BE 日志，检查 BE 节点是否启动成功。

   ```Bash
   cat be/log/be.INFO | grep heartbeat
   ```

   如果日志打印以下内容，则说明该 BE 节点启动成功：

   "I0810 16:18:44.487284 3310141 task_worker_pool.cpp:1387] Waiting to receive first heartbeat from frontend"

5. 在其他 BE 实例上重复以上步骤，即可启动新的 BE 节点。

> **说明**
>
> 在一个 StarRocks 集群中部署并添加至少 3 个 BE 节点后，这些节点将自动形成一个 BE 高可用集群。

## 第四步：（可选）启动 CN 服务

Compute Node（CN）是一种无状态的计算服务，本身不存储数据。您可以通过添加 CN 节点为查询提供额外的计算资源。您可以使用 BE 部署文件部署 CN 节点。CN 节点自 v2.4 版本起支持。

1. 进入先前准备好的 [StarRocks BE 部署文件](../deployment/prepare_deployment_files.md)所在路径，修改 CN 配置文件 **be/conf/cn.conf**。

   a. 如果任何在 [环境配置清单](../deployment/environment_configurations.md) 中提到的 CN 端口被占用，您必须在 CN 配置文件中为其分配其他可用端口。.

      ```YAML
      thrift_port = vvvv               # 默认值：9060
      be_http_port = xxxx              # 默认值：8040
      heartbeat_service_port = yyyy    # 默认值：9050
      brpc_port = zzzz                 # 默认值：8060
      ```

   b. 如需为集群启用 IP 地址访问，您必须在配置文件中添加配置项 `priority_networks`，为 CN 节点分配一个专有的 IP 地址（CIDR格式）。如需为集群启用 FQDN 访问，则可以忽略该配置项。.

      ```YAML
      priority_networks = x.x.x.x/x
      ```

      > **说明**
      >
      > 您可以在终端中运行 `ifconfig` 以查看当前实例拥有的 IP 地址。

   c. 如果您的实例安装了多个 JDK，并且您要使用 JDK 与环境变量 `JAVA_HOME` 中指定的不同，则必须在配置文件中添加配置项 `JAVA_HOME` 来指定所选该 JDK 的安装路径。

      ```YAML
      # 将 <path_to_JDK> 替换为所选 JDK 的安装路径。
      JAVA_HOME = <path_to_JDK>
      ```

   d.  由于大部分 CN 参数都继承自 BE 节点，您可以参考 [参数配置 - BE 配置项](../administration/Configuration.md#be-配置项) 了解更多 CN 高级配置项。

2. 启动 CN 节点。

   ```Bash
   ./be/bin/start_cn.sh --daemon
   ```

   > **注意**
   >
   > - 如需启用 FQDN 访问，在启动 CN 节点之前，请确保您已经在 **/etc/hosts** 中为所有实例分配了主机名。有关详细信息，请参考 [环境配置清单 - 主机名](../deployment/environment_configurations.md#主机名)。
   > - 启动 CN 节点时无需指定参数 `--host_type`。

3. 查看 CN 日志，检查 CN 节点是否启动成功。

   ```Bash
   cat be/log/cn.INFO | grep heartbeat
   ```

   如果日志打印以下内容，则说明该 CN 节点启动成功：

   "I0313 15:03:45.820030 412450 thrift_server.cpp:375] heartbeat has started listening port on 9050"

4. 在其他实例上重复以上步骤，即可启动新的 CN 节点。

## 第五步：搭建集群

当所有 FE、BE、CN 节点启动成功后，即可搭建 StarRocks 集群。

以下过程在 MySQL 客户端实例上执行。您必须安装 MySQL 客户端（5.5.0 或更高版本）。

1. 通过 MySQL 客户端连接到 StarRocks。您需要使用初始用户 `root` 登录，密码默认为空。

   ```Bash
   # 将 <fe_address> 替换为您连接的 FE 节点的 IP 地址（priority_networks）或 FQDN，
   # 并将 <query_port>（默认：9030）替换为您在 fe.conf 中指定的 query_port。
   mysql -h <fe_address> -P<query_port> -uroot
   ```

2. （可选）如果您部署了多个 Follower FE 节点，您可以通过执行以下 SQL 将额外的两个 Follower FE 节点添加至集群。

   ```SQL
   -- 将 <fe_address> 替换为您连接的 FE 节点的 IP 地址（priority_networks）或 FQDN，
   -- 并将 <edit_log_port>（默认：9010）替换为您在 fe.conf 中指定的 edit_log_port。
   ALTER SYSTEM ADD FOLLOWER "<fe2_address>:<edit_log_port>", "<fe3_address>:<edit_log_port>";
   ```

   > **说明**
   >
   > - 您可以通过一条 SQL 添加多个 Follower FE 节点。每对 `<fe_address>:<edit_log_port>` 代表一个 FE 节点。
   > - 如需添加更多的 Observer FE 节点，请执行 `ALTER SYSTEM ADD OBSERVER "<fe2_address>:<edit_log_port>"[, ...]`。有关详细说明，请参考 [ALTER SYSTEM - FE](../sql-reference/sql-statements/Administration/ALTER%20SYSTEM.md)。

3. 执行以下 SQL 查看 FE 节点状态。

   ```SQL
   SHOW PROC '/frontends'\G
   ```

   示例：

   ```Plain
   MySQL [(none)]> SHOW PROC '/frontends'\G
   *************************** 1. row ***************************
                 Name: x.x.x.x_9010_1660119137253
                   IP: x.x.x.x
          EditLogPort: 9010
             HttpPort: 8030
            QueryPort: 9030
              RpcPort: 9020
                 Role: FOLLOWER
             IsMaster: true
            ClusterId: 58958864
                 Join: true
                Alive: true
    ReplayedJournalId: 30602
        LastHeartbeat: 2022-08-11 20:34:26
             IsHelper: true
               ErrMsg: 
            StartTime: 2022-08-10 16:12:29
              Version: 2.3.0-a9bdb09
   1 row in set (0.01 sec)
   ```

   - 如果字段 `Alive` 为 `true`，说明该 FE 节点正常启动并加入集群。
   - 如果字段 `Role` 为 `FOLLOWER`，说明该 FE 节点有资格被选为 Leader FE 节点。
   - 如果字段 `Role` 为 `LEADER`，说明该 FE 节点为 Leader FE 节点。

4. 添加 BE 节点至集群。

   ```SQL
   -- 将 <be_address> 替换为 BE 节点的 IP 地址（priority_networks）或 FQDN，
   -- 并将 <heartbeat_service_port>（默认：9050）替换为您在 be.conf 中指定的 heartbeat_service_port。
   ALTER SYSTEM ADD BACKEND "<be_address>:<heartbeat_service_port>", "<be2_address>:<heartbeat_service_port>", "<be3_address>:<heartbeat_service_port>";
   ```

   > **说明**
   >
   > 您可以通过一条 SQL 添加多个 BE 节点。每对 `<be_address>:<heartbeat_service_port>` 代表一个 BE 节点。

5. 执行以下 SQL 查看 BE 节点状态。

   ```SQL
   SHOW PROC '/backends'\G
   ```

   示例：

   ```Plain
   MySQL [(none)]> SHOW PROC '/backends'\G
   *************************** 1. row ***************************
               BackendId: 10036
                 Cluster: default_cluster
                      IP: x.x.x.x
           HeartbeatPort: 9050
                  BePort: 9060
                HttpPort: 8040
                BrpcPort: 8060
           LastStartTime: 2022-08-10 17:39:01
           LastHeartbeat: 2022-08-11 20:34:31
                   Alive: true
    SystemDecommissioned: false
   ClusterDecommissioned: false
               TabletNum: 0
        DataUsedCapacity: .000 
           AvailCapacity: 1.000 B
           TotalCapacity: .000 
                 UsedPct: 0.00 %
          MaxDiskUsedPct: 0.00 %
                  ErrMsg: 
                 Version: 2.3.0-a9bdb09
                  Status: {"lastSuccessReportTabletsTime":"N/A"}
       DataTotalCapacity: .000 
             DataUsedPct: 0.00 %
                CpuCores: 16
   ```

   如果字段 `Alive` 为 `true`，说明该 BE 节点正常启动并加入集群。

6. （可选）添加 CN 节点至集群。

   ```SQL
   -- 将 <cn_address> 替换为 CN 节点的 IP 地址（priority_networks）或 FQDN，
   -- 并将 <heartbeat_service_port>（默认：9050）替换为您在 cn.conf 中指定的 heartbeat_service_port。
   ALTER SYSTEM ADD COMPUTE NODE "<cn_address>:<heartbeat_service_port>", "<cn2_address>:<heartbeat_service_port>", "<cn3_address>:<heartbeat_service_port>";
   ```

   > **说明**
   >
   > 您可以通过一条 SQL 添加多个 CN 节点。每对 `<cn_address>:<heartbeat_service_port>` 代表一个 CN 节点。

7. （可选）执行以下 SQL 查看 CN 节点状态。

   ```SQL
   SHOW PROC '/compute_nodes'\G
   ```

   示例：

   ```Plain
   MySQL [(none)]> SHOW PROC '/compute_nodes'\G
   *************************** 1. row ***************************
           ComputeNodeId: 10003
                      IP: x.x.x.x
           HeartbeatPort: 9550
                  BePort: 9060
                HttpPort: 8540
                BrpcPort: 8560
           LastStartTime: 2023-03-13 15:11:13
           LastHeartbeat: 2023-03-13 15:11:13
                   Alive: true
    SystemDecommissioned: false
   ClusterDecommissioned: false
                  ErrMsg: 
                 Version: 2.5.2-c3772fb
   1 row in set (0.00 sec)
   ```

   如果字段 `Alive` 为 `true`，说明该 CN 节点正常启动并加入集群。

## 停止 StarRocks 集群

您可以通过在相应实例上运行以下命令来停止 StarRocks 集群。

- 停止 FE 节点。

  ```Bash
  ./fe/bin/stop_fe.sh --daemon
  ```

- 停止 BE 节点。

  ```Bash
  ./be/bin/stop_be.sh --daemon
  ```

- 停止 CN 节点。

  ```Bash
  ./be/bin/stop_cn.sh --daemon
  ```

## 故障排除

如果启动 FE、BE 或 CN 节点失败，尝试以下步骤来发现问题：

- 如果 FE 节点没有正常启动，您可以通过查看 **fe/log/fe.warn.log** 中的日志来确定问题所在。

  ```Bash
  cat fe/log/fe.warn.log
  ```

  确定并解决问题后，您首先需要终止当前 FE 进程，删除现有的 **meta** 路径，新建元数据存储路径，然后以正确的配置重启该 FE 节点。

- 如果 BE 节点没有正常启动，您可以通过查看 **be/log/be.WARNING** 中的日志来确定问题所在。

  ```Bash
  cat be/log/be.WARNING
  ```

  确定并解决问题后，您首先需要终止当前 BE 进程，删除现有的 **storage** 路径，新建数据存储路径，然后以正确的配置重启该 BE 节点。

- 如果 CN 节点没有正常启动，您可以通过查看 **be/log/cn.WARNING** 中的日志来确定问题所在。

  ```Bash
  cat be/log/cn.WARNING
  ```

  确定并解决问题后，您首先需要终止当前 CN 进程，然后以正确的配置重启该 CN 节点。

## 下一步

成功部署 StarRocks 集群后，您可以参考 [部署后设置](../deployment/post_deployment_setup.md) 以获取有关初始管理措施的说明。
