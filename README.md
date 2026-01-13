# Ansible-Mysql

**Ansible-Mysql** 是一个用于 **MySQL 集群自动化部署与高可用管理** 的 Ansible Playbook/Role 集合，目的是为了实现有关Mysql的部署自动化和自动化分析工具，可以帮助运维团队能够简化相关的工作流程，并且使用该工具能够快速排查问题，当前已实现：

- 自动化安装二进制 MySQL（支持 8.0 系列，可通过变量切换版本）

- 自动化配置基于 GTID 的 MySQL 主从复制拓扑

- 自动化部署 Orchestrator（包含 Agent 与 Server 两种模式），实现高可用切换与拓扑管理

项目结构遵循 Ansible 官方推荐的 **多 Role 分层结构**，同时预留了变量与 Role 扩展点，方便后续继续增加更多 MySQL 自动化功能（如备份、审计、参数调优、慢日志分析等）。


[![License](https://img.shields.io/badge/license-Apache2-lightgrey)](./LICENSE)
[![Qt](https://img.shields.io/badge/Ansible-41b883?logo=ansible)](https://www.qt.io)
[![Language](https://img.shields.io/badge/language-Python-blue?logo=c%2B%2B)](https://isocpp.org)
[![Platforms](https://img.shields.io/badge/platforms-Linux-lightgrey)](#)

当前自动化工作主要使用ansible和python相互结合实现，能够做到批量多主机操作，方便管理多个mysql节点，支持的平台为 **Linux**，包括 Ubuntu/Debian 和 CentOS/Redhat，

</div>

> 注意：本项目处于积极开发阶段，许多功能仍在完善中。欢迎贡献代码——如果你希望参与开发，请联系作者。😉

---

## 目录结构

核心目录与文件说明（简化展示）：

- `dependencies.yml`：控制端依赖初始化（如 rsync）

- `site.yml`：主入口 playbook，按阶段编排各 Role

- `requirements.yml`：Ansible Galaxy 集合依赖（community.mysql）

- `group_vars/`

  - `mysql_install.yml`：MySQL 安装相关变量（版本、路径、root 密码等）

  - `mysql_replication.yml`：主从复制架构与凭据配置

  - `orchestrator.yml`：Orchestrator 部署与后端数据库、VIP 等配置

- `inventory/`

  - `mysql_install_hosts`：用于 MySQL 安装与复制配置的主机清单（组名一般为 `mysql_nodes`）

  - `mysql_replication_hosts`：如需与安装分离，可独立用于复制配置

  - `orchestrator_hosts`：Orchestrator 管理端与被管理节点清单（组名示例：`mysql_nodes`、`orc_manager`）

- `roles/`

  - `mysql_install/`：MySQL 安装 Role

  - `mysql_replication/`：MySQL 主从复制 Role

  - `orchestrator/`：Orchestrator 安装与配置 Role

---

## 支持平台与前置条件

### 支持的 Linux 发行版

本项目已按 Ansible 条件分支适配以下平台：

- Debian 家族

  - Debian 11/12

  - Ubuntu 18.04 / 20.04 / 22.04

- RedHat 家族

  - CentOS 7 / 8 / Stream

  - RedHat Enterprise Linux 7 / 8 / 9

  - 以及兼容的 Rocky Linux / AlmaLinux 等

> 实际部署前建议在测试环境中按「平台测试方案」章节进行验证。

### 控制端（Ansible 主机）要求

- 已安装 Ansible（推荐 2.13+）

- 可以通过 SSH 密钥或密码访问各目标节点

- Python 3 环境可用

- 能访问 MySQL 官方 CDN（除非你提供本地安装包 `local_package_path`）

### 受管节点（MySQL / Orchestrator 节点）要求

- 已安装 Python（一般 Linux 默认自带）

- 可以通过 `sudo` 或 root 提权

- 出站网络能够访问 MySQL 官方 CDN（或者通过控制端推送本地安装包）

---

## 安装依赖

1. 安装 Ansible Galaxy 集合依赖（community.mysql）：

```bash
ansible-galaxy install -r requirements.yml
```

2. 在控制端初始化依赖（例如 rsync）：

```bash
ansible-playbook dependencies.yml
```

---

## 核心功能说明

### 1. MySQL 自动化安装（role: `mysql_install`）

**入口：**

- playbook：`site.yml` 中的「MySQL 节点部署」阶段

- 主机组：`mysql_nodes`

- 标签：`mysql_install`, `mysql`

**主要能力：**

- 在控制端准备 MySQL 安装包（本地/远程下载，支持 CDN 与本地路径）

- 根据 `ansible_os_family` 自动安装平台依赖包：

  - Debian/Ubuntu 使用 `apt`

  - CentOS/RedHat 使用 `yum`

- 检查目标端口是否占用，避免覆盖已有实例

- 使用 `synchronize`（rsync）高效分发安装包到各节点

- 在目标节点本地解压、移动至正式目录，创建 MySQL 用户与数据目录

- 渲染 `my.cnf`（模板：`roles/mysql_install/templates/my.cnf.j2`）

- 初始化 MySQL 数据目录，并配置 systemd 服务（模板：`roles/mysql_install/templates/mysql.service.j2`）

- 启动 MySQL 服务，设置 root 密码和远程访问权限

- 可选清理控制端上的安装包（通过 `cleanup_local_strategy` 控制）

**关键变量（详见 `group_vars/mysql_install.yml`）：**

- 版本与安装源：

  - `mysql_major_version`

  - `mysql_version`

  - `mysql_arch`

  - `mysql_download_url`

  - `local_package_path`

- 路径与目录：

  - `install_base_dir`

  - `mysql_install_dir`

  - `mysql_data_dir`

  - `mysql_config_path`

- 运行参数：

  - `mysql_port`

  - `mysql_user`

  - `mysql_group`

  - `mysql_root_password`

  - `mysql_auth_plugin`

  - `mysql_max_connections` / `mysql_char_set` / `mysql_storage_engine` 等

- 控制端清理策略：

  - `cleanup_local_strategy`（是否删除控制端下载的安装包）

---

### 2. MySQL 主从复制（role: `mysql_replication`）

**入口：**

- playbook：`site.yml` 中的「MySQL 节点初始化与复制配置」阶段

- 主机组：`mysql_nodes`

- 标签：`mysql_replication`, `mysql`

**主要能力：**

- 安装 Python MySQL 驱动（PyMySQL），以支持 `community.mysql` 集合：

  - Debian/Ubuntu：`python3-pymysql`

  - CentOS/RedHat：`python3-PyMySQL`

- 在 `my.cnf` 的 `[mysqld]` 段落中自动追加复制相关配置：

  - 自动生成唯一 `server_id`（基于 `server_id_prefix` 与主机索引）

  - 开启 binlog：`log_bin`, `binlog_format=ROW`

  - 关闭 DNS 反查：`skip_name_resolve=1`

  - 开启 GTID 与强一致性：`gtid_mode=ON`, `enforce_gtid_consistency=ON`

  - 认证插件兼容：`default_authentication_plugin=mysql_native_password`

- 重启 MySQL 以应用新配置，并校验 `GTID_MODE` 是否为 `ON`

- 在指定主库 `initial_master_ip` 上创建复制专用账号 `repl_user`

- 在所有从库上：

  - 检查当前复制状态

  - 如未配置复制，则执行 `CHANGE MASTER TO ... MASTER_AUTO_POSITION = 1`

  - 启动复制线程（IO/SQL）

**关键变量（详见 `group_vars/mysql_replication.yml`）：**

- 架构与 server_id：

  - `initial_master_ip`：初始主库 IP

  - `server_id_prefix`：用于生成唯一 `server_id`

- 认证信息：

  - `mysql_root_password`

  - `repl_user`

  - `repl_password`

- 配置文件路径：

  - `mysql_config_file`（通常为 `/etc/my.cnf`）

  - `mysql_log_bin`（如 `mysql-bin`）

---

### 3. Orchestrator 高可用（role: `orchestrator`）

Orchestrator 部署分为两部分：

1. 在 MySQL 节点上配置 Agent 相关参数与监控账号

2. 在管理端部署 Orchestrator Server 与 Web 界面

**入口：**

- playbook：`site.yml` 中的两个阶段：

  - 「Orchestrator Agent 配置 (MySQL节点)」

  - 「Orchestrator Server 部署 (管理机)」

- 主机组：

  - `mysql_nodes`（Agent 模式）

  - `orc_manager`（Server 模式）

- 标签：

  - `orc_agent`, `orchestrator`（Agent）

  - `orc_server`, `orchestrator`（Server）

**主要能力（Agent 模式）：**

- 等待 SSH 连接，确保节点可达

- 在 `my.cnf` 的 `[mysqld]` 段落中添加：

  - `report_host = <ansible_host>`

  - `skip-name-resolve`

- 创建 Orchestrator 拓扑监控账号：

  - 关闭 binlog 日志：`SET SQL_LOG_BIN=0`

  - 创建用户 `orc_topology_user` 并赋予监控所需权限

  - 恢复 binlog 记录

**主要能力（Server 模式）：**

- 初始化 Orchestrator 后端元数据库：

  - 创建 `orchestrator` 库

  - 创建后端账号 `backend_mysql_user`，并授权 `orchestrator.*`

- 清洗旧拓扑数据（如有历史残留）：

  - TRUNCATE `database_instance` 等核心表

- 下载并安装 Orchestrator 二进制：

  - 从 `orc_pkg_url` 下载 tar.gz 包

  - 解压到临时目录，自动查找 `orchestrator` 二进制

  - 拷贝到 `orc_install_dir`，并复制 `resources` 目录（如存在）

  - 添加到 `/etc/profile` 与 `/usr/bin` 软链接

- 渲染配置与服务：

  - `orchestrator.service`（systemd 服务文件）

  - `orchestrator.conf.json`（核心配置）

  - `orc_vip_hook.sh`（VIP 高可用脚本）

- 启动并启用 Orchestrator：

  - `systemd` 重载并启动服务

  - 等待 Web 端口（默认 3000）可用

  - 调用 Web API 触发拓扑发现

**关键变量（详见 `group_vars/orchestrator.yml`）：**

- 监控账号：

  - `orc_topology_user`

  - `orc_topology_pass`

- 业务 MySQL root：

  - `biz_mysql_root_password`

- 后端元数据库配置：

  - `backend_mysql_host`

  - `backend_mysql_port`

  - `backend_mysql_root_pass`

  - `backend_mysql_user`

  - `backend_mysql_pass`

- VIP 高可用参数：

  - `vip_address`

  - `vip_mask`

  - `vip_interface`

  - `initial_master_ip`

- Orchestrator 版本与安装路径：

  - `orc_version`

  - `orc_pkg_url`

  - `orc_install_dir`

---

## 使用示例

### 1. 只执行 MySQL 安装

```bash
ansible-playbook \
  -i inventory/mysql_install_hosts \
  site.yml \
  --tags "mysql_install" \
  -e "@group_vars/mysql_install.yml"
```

### 2. 只执行 MySQL 主从复制

```bash
ansible-playbook \
  -i inventory/mysql_replication_hosts \
  site.yml \
  --tags "mysql_replication" \
  -e "@group_vars/mysql_replication.yml"
```

### 3. 只部署 Orchestrator（Agent + Server）

```bash
ansible-playbook \
  -i inventory/orchestrator_hosts \
  site.yml \
  --tags "orchestrator" \
  -e "@group_vars/orchestrator.yml"
```

### 4. 跳过某个 Role（例如跳过 MySQL 安装）

```bash
ansible-playbook \
  -i inventory/mysql_install_hosts \
  site.yml \
  --skip-tags "mysql_install"
```

> 提示：可以组合使用 `--tags` 和 `--skip-tags` 精确控制执行范围，例如只在部分节点上调整复制配置等。

---

## 平台测试方案（Linux）

为了保证此 Playbook 在不同 Linux 发行版上的稳定性，建议按照以下矩阵与步骤进行测试。

### 测试矩阵

建议选取以下最小覆盖：

- Ubuntu 20.04 LTS

- Debian 11

- CentOS 7

- Rocky Linux / AlmaLinux / RHEL 8 或 9（任选一款代表 RedHat 家族）

每类平台建议至少准备：

- 1 台作为初始主库（`initial_master_ip`）

- 1–2 台作为从库

- 1 台作为 Orchestrator 管理端（也可以与部分 MySQL 节点共用，视资源情况而定）

### 通用测试步骤

1. **准备基础环境**

   - 为所有测试虚机配置好 SSH 访问和 `sudo` 权限

   - 在控制端配置好 `inventory` 中的主机名/IP 与分组（`mysql_nodes`、`orc_manager` 等）

1. **环境连通性与事实采集测试**

   ```bash
   ansible -i inventory/mysql_install_hosts all -m ping
   ansible -i inventory/mysql_install_hosts all -m setup -a 'filter=ansible_os_family'
   ```

   - 确认各节点的 `ansible_os_family` 返回为 `Debian` 或 `RedHat`，以匹配 playbook 中的条件判断。

3. **单 Role 功能验证**

   按顺序、按平台执行：

   - MySQL 安装：

```bash
ansible-playbook -i inventory/mysql_install_hosts site.yml \
--tags "mysql_install" -e "@group_vars/mysql_install.yml"
```

     验证点：

     - MySQL 安装目录是否正确（`mysql_install_dir`）

     - 服务是否已通过 systemd 启动并监听 `mysql_port`

     - 能否以 root 账号远程连接并验证密码

   - 主从复制：

```bash
ansible-playbook -i inventory/mysql_replication_hosts site.yml \
--tags "mysql_replication" -e "@group_vars/mysql_replication.yml"
```

     验证点：

     - 所有从库 `SHOW REPLICA STATUS\G` 中 IO/SQL 线程是否为 `Yes`

     - `GTID_MODE` 是否为 `ON`，复制是否使用 `AUTO_POSITION = 1`

   - Orchestrator：

```bash
ansible-playbook -i inventory/orchestrator_hosts site.yml \
--tags "orchestrator" -e "@group_vars/orchestrator.yml"
```

     验证点：
     
     - Orchestrator Web UI 是否能访问（默认 3000 端口）

     - 拓扑页面是否正确展示主从关系

     - 模拟主库宕机后，是否能按预期完成主从切换（如启用 VIP 脚本时，VIP 是否漂移）

4. **幂等性测试**

   在上述步骤完成后，至少在每个平台上重复执行一次相同命令，确认：

   - 第二次执行时不应产生不必要的 `changed`（除非确实有状态变更）

   - 重要操作（如用户创建、数据库初始化）应通过 `IF NOT EXISTS` 等保证幂等

5. **异常场景测试（可选）

   - 端口已被占用时，确认 playbook 会给出合理警告而非破坏现有实例

   - CDN 不可达时，使用本地包路径 `local_package_path` 能否顺利安装

   - Orchestrator 后端数据库权限不足时，报错提示是否清晰

---

## 扩展与定制

为了方便后续继续增加更多与 MySQL 相关的自动化能力，推荐如下扩展方式：

1. **新增 Role**

   - 在 `roles/` 下新增独立 Role，例如：

     - `mysql_backup`：备份/恢复

     - `mysql_tuning`：参数调优

     - `mysql_audit`：安全审计

   - 在 `site.yml` 中增加新的阶段，并通过 `tags` 对不同功能进行隔离。

1. **扩展变量**

   - 在 `group_vars/` 下为每个功能分组维护独立的变量文件

   - 通过 `-e "@group_vars/xxx.yml"` 显式加载，避免变量互相污染

   - 对敏感信息推荐使用 Ansible Vault 进行加密

1. **模板与脚本定制**

   - 修改 `roles/*/templates/` 下的 Jinja2 模板即可定制配置文件

   - 对运维脚本（如 VIP 切换脚本）建议提供参数化变量，保持复用性

1. **CI / 自动化测试（建议）**

   - 可结合 Molecule + Docker/Vagrant 为每个平台编写独立的测试场景

   - 在 CI（如 GitHub Actions）中为典型发行版运行 Molecule 场景，自动验证 Playbook 的幂等性与收敛性

---

## 贡献与问题反馈

- 如需在现有基础上增加新的 MySQL 自动化功能，建议：

  - 先在 `roles/` 下创建新 Role，再在 `site.yml` 中编排执行顺序与标签

  - 为新功能在 README 中补充对应小节与变量说明

- 如在使用过程中遇到问题，建议收集以下信息后提交到代码仓库 Issue：

  - Ansible 版本、目标操作系统与版本

  - 执行命令、`inventory` 片段与相关 `group_vars` 内容（注意去除敏感信息）

  - 完整的 Ansible 输出（加上 `-vvv` 级别更有助于排查）
