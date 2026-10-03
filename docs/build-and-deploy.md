# SeaTunnel Web 打包与部署文档

> 适用分支：`2.3.13-teaboy`（适配 SeaTunnel **2.3.13**）
> 产物版本：`apache-seatunnel-web-1.0.3-SNAPSHOT`
> 文档中所有命令、产物大小、文件数量均为实际执行验证过的结果。

## 一、环境要求

|   项目    |             要求              |                     说明                     |
|---------|-----------------------------|--------------------------------------------|
| JDK     | **21**（本分支已适配）              | 见下方「JDK 版本说明」                              |
| Maven   | 3.8.4+（仓库自带 `mvnw` 为 3.8.4） | 本机实测 3.9.16 可用                             |
| Node.js | **无需预装**                    | 构建时由 `frontend-maven-plugin` 自动下载 v16.16.0 |
| npm     | 无需预装                        | 自动下载 8.19.2                                |

### JDK 版本说明

本分支已将构建工具链升级为兼容 JDK 21 的版本：

|          工具           |   版本    |   原版本   |
|-----------------------|---------|---------|
| lombok                | 1.18.36 | 1.18.24 |
| spotless-maven-plugin | 2.43.0  | 2.29.0  |
| google-java-format    | 1.19.2  | 1.7     |

> **重要**：升级后 **JDK 8 已无法构建本分支**。
> spotless 2.43.0 的字节码是 Java 11（major=55），JDK 8 无法加载。
> 且 spotless 在 JVM 21 上强制要求 google-java-format ≥ 1.17.0，
> 而 GJF 1.8+ 又要求 JDK 11+，因此不存在同时兼容 JDK 8 与 JDK 21 的组合。
>
> 编译产物仍为 **Java 8 字节码**（`pom.xml` 中 `java.version=1.8` 未改动），
> 运行时兼容 JDK 8，与构建时使用的 JDK 版本无关。

## 二、打包命令

### 2.1 标准打包命令（国内网络环境，推荐）

在项目根目录 `seatunnel-web/` 下执行：

**Linux / macOS / Git Bash：**

```bash
export npm_config_registry=https://registry.npmmirror.com
export npm_config_maxsockets=16

mvn -B clean package -DskipTests -Pci \
  -DnodeDownloadRoot=https://cdn.npmmirror.com/binaries/node/ \
  -DnpmDownloadRoot=https://registry.npmmirror.com/npm/-/
```

**Windows PowerShell：**

```powershell
$env:npm_config_registry = "https://registry.npmmirror.com"
$env:npm_config_maxsockets = "16"

mvn -B clean package -DskipTests -Pci `
  "-DnodeDownloadRoot=https://cdn.npmmirror.com/binaries/node/" `
  "-DnpmDownloadRoot=https://registry.npmmirror.com/npm/-/"
```

**Windows CMD：**

```bat
set npm_config_registry=https://registry.npmmirror.com
set npm_config_maxsockets=16

mvn -B clean package -DskipTests -Pci ^
  -DnodeDownloadRoot=https://cdn.npmmirror.com/binaries/node/ ^
  -DnpmDownloadRoot=https://registry.npmmirror.com/npm/-/
```

实测耗时约 **4 分 10 秒**（34 个模块全部 SUCCESS，依赖已缓存时更快）。

### 2.2 官方脚本（海外网络 / 无需镜像）

网络可直连 nodejs.org 与 npmjs.org 时，可直接使用仓库自带脚本：

```bash
sh build.sh code        # Linux / macOS
build.cmd code          # Windows
```

其内部执行的就是 `mvnw clean package -DskipTests -Pci`，
并把产物 zip 移动到项目根目录。

### 2.3 各参数说明

|           参数            |                       作用                        |
|-------------------------|-------------------------------------------------|
| `-DskipTests`           | 跳过测试，加快打包                                       |
| `-Pci`                  | 激活 `ci` profile（`activeByDefault=true`，即使不写也生效） |
| `-DnodeDownloadRoot`    | 指定 Node.js 下载镜像根地址                              |
| `-DnpmDownloadRoot`     | 指定 npm 下载镜像根地址                                  |
| `npm_config_registry`   | 环境变量，覆盖 `seatunnel-ui/.npmrc` 里的 npm registry   |
| `npm_config_maxsockets` | 环境变量，覆盖 `.npmrc` 里的 `maxsockets=1`（串行下载）        |

#### 镜像地址拼接规则（易错点）

`frontend-maven-plugin` 的 URL 拼接方式**两者不同**，填错会 404：

|         参数         |                  拼接规则                   |                    因此应填                    |
|--------------------|-----------------------------------------|--------------------------------------------|
| `nodeDownloadRoot` | `{root}v16.16.0/win-x64/node.exe`       | `https://cdn.npmmirror.com/binaries/node/` |
| `npmDownloadRoot`  | `{root}npm-8.19.2.tgz`（**不含** `npm/-/`） | `https://registry.npmmirror.com/npm/-/`    |

> `npmDownloadRoot` 若误填为 `https://registry.npmmirror.com/`，
> 实际请求会变成 `https://registry.npmmirror.com/npm-8.19.2.tgz` 并返回 **404**，
> 构建报 `Could not download npm: Got error code 404 from the server.`

#### 为什么必须走镜像

`seatunnel-ui/.npmrc` 中 registry 固定为 `registry.npmjs.org`，且 `maxsockets=1`。
实测对比：

|                 源                  |         结果          |
|------------------------------------|---------------------|
| `nodejs.org` 直连                    | 下载 `node.exe` 卡死无响应 |
| `registry.npmjs.org` 的 npm tgz     | 无响应                 |
| `registry.npmjs.org` 取 vue 元数据     | 6.53 秒              |
| `registry.npmmirror.com` 取 vue 元数据 | 1.18 秒              |

项目有 **2632 个传递依赖**，在 `maxsockets=1`（串行）下用官方源会非常慢，
故通过环境变量提升并发到 16。

## 三、构建产物

打包成功后，产物位于 `seatunnel-web-dist/target/`：

|                      文件                      |    大小    |
|----------------------------------------------|----------|
| `apache-seatunnel-web-1.0.3-SNAPSHOT.tar.gz` | 394.1 MB |
| `apache-seatunnel-web-1.0.3-SNAPSHOT.zip`    | 394.3 MB |
| `apache-seatunnel-web-1.0.3-SNAPSHOT/`（目录形式） | —        |

> 压缩包内含一层同名目录（assembly 配置 `includeBaseDirectory=true`），
> 解压后进入 `apache-seatunnel-web-1.0.3-SNAPSHOT/` 才是真正的部署根目录。

### 3.1 部署包目录结构

```text
apache-seatunnel-web-1.0.3-SNAPSHOT/
├── bin/            # 启动脚本（2 个文件）
│   ├── seatunnel-backend-daemon.sh    # 启停脚本
│   └── download_datasource.sh         # 数据源插件下载脚本
├── conf/           # 配置文件（3 个文件）
│   ├── application.yml
│   ├── connector-datasource-mapper.yaml
│   └── logback-spring.xml
├── libs/           # 后端依赖 jar（158 个文件）
├── datasource/     # 数据源插件 jar（20 个文件）
├── script/         # 数据库初始化脚本（4 个文件）
│   ├── init_sql.sh
│   ├── seatunnel_server_env.sh
│   ├── seatunnel_server_mysql.sql
│   └── seatunnel_server_h2.sql
├── ui/             # 前端构建产物（125 个文件）
│   ├── index.html
│   ├── assets/
│   └── logo-favicon.png
├── licenses/
├── LICENSE
├── NOTICE
└── DISCLAIMER
```

### 3.2 产物校验

已验证打包产物中的 SeaTunnel 依赖版本正确：

```bash
# libs/ 下的 SeaTunnel 制品应全部为 2.3.13
ls libs/ | grep '^seatunnel-'
```

预期包含 `seatunnel-api-2.3.13.jar`、`seatunnel-engine-client-2.3.13.jar`、
`seatunnel-engine-common-2.3.13.jar`、`seatunnel-engine-core-2.3.13.jar`、
`seatunnel-transforms-v2-2.3.13.jar` 等共 **20 个 2.3.13 制品**，
且 `find . -name "*2.3.11*"` 结果为 **0**（无旧版本残留）。

> 注意：`seatunnel-engine-common-2.3.13.jar` 必须存在。
> SeaTunnel 2.3.13 中 `JobStatus`、`JobResult` 两个类
> 已从 `engine.core.job` 包迁移到 `engine.common.job` 包，
> 该 jar 正是它们的新归属。

## 四、部署步骤

### 4.1 前置条件

部署机上需先安装好 **SeaTunnel 2.3.13** 服务端（Zeta 引擎），
后续步骤依赖它的安装目录 `SEATUNNEL_HOME`。

### 4.2 解压部署包

```bash
tar -zxvf apache-seatunnel-web-1.0.3-SNAPSHOT.tar.gz
cd apache-seatunnel-web-1.0.3-SNAPSHOT
```

### 4.3 初始化数据库

编辑 `script/seatunnel_server_env.sh`，填写数据库连接信息：

```bash
export HOSTNAME="127.0.0.1"
export PORT="3306"
export USERNAME="root"
export PASSWORD="123456"
```

执行初始化（需本机已安装 mysql 客户端）：

```bash
sh script/init_sql.sh
```

该脚本会执行 `seatunnel_server_mysql.sql` 建表。

### 4.4 修改配置文件 `conf/application.yml`

**① 数据库连接**（默认值需按实际环境修改）：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/seatunnel?useSSL=false&useUnicode=true&characterEncoding=utf-8&allowMultiQueries=true&allowPublicKeyRetrieval=true
    username: root
    password: 123456
```

**② JWT 密钥（必填）**

`jwt.secretKey` 默认为空，源码注释明确标注 `please add key when deploy`，
**不填写将无法正常登录**：

```yaml
jwt:
  expireTime: 86400
  secretKey: <填写足够长的随机字符串>
```

**③ 服务端口**（默认 `8801`，按需修改）：

```yaml
server:
  port: 8801
```

### 4.5 拷贝必需的外部配置文件

`conf/` 目录**只包含** `*.yml`、`*.yaml`、`*.xml` 三类文件
（由 assembly 配置决定），因此以下两个文件**必须手动拷贝**，否则功能不完整：

```bash
# ① SeaTunnel 客户端配置
cp $SEATUNNEL_HOME/config/hazelcast-client.yaml ./conf/

# ② 连接器插件映射表
cp $SEATUNNEL_HOME/connectors/plugin-mapping.properties ./conf/
```

> `plugin-mapping.properties` 是**连接器插件发现**的关键文件，
> 缺失会导致数据源插件无法识别。
> 源码中该文件位于 `seatunnel-server/seatunnel-app/src/main/resources/`，
> 但因扩展名是 `.properties`，不在 assembly 的 include 范围内，
> 故不会被打进部署包。

### 4.6 设置环境变量

启动脚本会**强制校验** `SEATUNNEL_HOME` 与 `JAVA_HOME`，
任一未设置都会直接退出：

```bash
export SEATUNNEL_HOME=/path/to/apache-seatunnel-2.3.13
export JAVA_HOME=/path/to/jdk
```

> `SEATUNNEL_HOME` 必须指向 **SeaTunnel 2.3.13** 的安装目录，
> 且建议部署在与 SeaTunnel Zeta 服务端相同的路径下。

### 4.7 启动服务

```bash
sh bin/seatunnel-backend-daemon.sh start    # 启动
sh bin/seatunnel-backend-daemon.sh status   # 查看状态
sh bin/seatunnel-backend-daemon.sh stop     # 停止
```

脚本细节：

- 主类：`org.apache.seatunnel.app.SeatunnelApplication`
- 默认 JVM 参数：`-Xms1g -Xmx1g -Xmn512m`（可通过 `JAVA_OPTS` 环境变量追加覆盖）
- CLASSPATH：`conf/`、`libs/*`、`datasource/*`
- 日志目录：`logs/`（自动创建），主日志 `logs/seatunnel.out`，GC 日志 `logs/gc.log`

启动后访问 `http://<部署机IP>:8801`。

### 4.8 关于 `download_datasource.sh`

该脚本用于从 Maven 仓库下载数据源插件 jar（默认版本 `1.0.0`）。

**使用 `-Pci` 打包时无需执行**——部署包的 `datasource/` 目录
已自带 20 个与当前代码同版本（`1.0.3-SNAPSHOT`）的插件 jar。
仅当需要额外补充其他版本的数据源插件时才使用：

```bash
sh bin/download_datasource.sh 1.0.3-SNAPSHOT
```

## 五、常见问题

### 5.1 构建卡在下载 node.exe

**现象**：日志停在
`Downloading https://nodejs.org/dist/v16.16.0/win-x64/node.exe` 不再前进。

**原因**：直连 nodejs.org 下载超时。

**解决**：使用 2.1 节的镜像参数 `-DnodeDownloadRoot=https://cdn.npmmirror.com/binaries/node/`。
下载成功后 node 会缓存到本地 Maven 仓库
（`com/github/eirslett/node/16.16.0/`），后续构建不再重复下载。

### 5.2 构建报 `Could not download npm: Got error code 404`

**原因**：`-DnpmDownloadRoot` 填写的地址少了 `npm/-/` 路径段。
插件按 `{root}npm-8.19.2.tgz` 拼接，
填 `https://registry.npmmirror.com/` 会得到 404 的地址。

**解决**：改为 `-DnpmDownloadRoot=https://registry.npmmirror.com/npm/-/`。

### 5.3 `npm install` 极慢

**原因**：`seatunnel-ui/.npmrc` 中 `maxsockets=1`（串行下载）+ 2632 个传递依赖。

**解决**：设置环境变量 `npm_config_maxsockets=16` 与
`npm_config_registry=https://registry.npmmirror.com`。
用环境变量而非改文件，可避免污染仓库。

### 5.4 spotless 报 pom.xml 整个文件每行都违规

**现象**：报错形如 `@@ -1,1620 +1,1620 @@`，全文件每行都是差异。

**原因**：文件行尾被从 CRLF 转成了 LF。本仓库 `core.autocrlf=true`
且 `.gitattributes` 为空，工作区文件应为 CRLF，spotless 在 Windows 上期望 CRLF。

**特别注意**：`git diff` 会把 LF/CRLF 视为等价，**只显示内容改动、完全隐藏行尾破坏**，
所以 diff 看着干净也会构建失败。

**排查**：用 `tr` 精确统计 CR 字节数（`grep -c $'\r'` 在 Git Bash 下结果不可靠）：

```bash
tr -cd '\r' < pom.xml | wc -c    # 结果应等于文件行数；为 0 则说明行尾被破坏
```

**避免**：在 Git Bash 下**不要用 `sed -i`** 改文件，它会把整个文件行尾转成 LF。
改用编辑器的查找替换，或 `perl -pi -e 's/\r?\n$/\r\n/'`（保持 CRLF）。

**修复**：

```bash
# 批量把被破坏的文件恢复为 CRLF
for f in $(git diff HEAD --name-only); do
  [ -f "$f" ] || continue
  [ "$(tr -cd '\r' < "$f" | wc -c)" -eq 0 ] && perl -pi -e 's/\r?\n$/\r\n/' "$f"
done
```

### 5.5 构建报 lombok 相关 `NoSuchFieldError`

**现象**：
`java.lang.NoSuchFieldError: Class com.sun.tools.javac.tree.JCTree$JCImport does not have member field 'com.sun.tools.javac.tree.JCTree qualid'`

**原因**：lombok 1.18.24 不支持 JDK 21。

**解决**：本分支已升级到 lombok 1.18.36，正常情况下不会出现。
若仍遇到，检查 `pom.xml` 中 `<lombok.version>` 是否为 `1.18.36`。

### 5.6 构建报 spotless `InvocationTargetException`

**现象**：
`InvocationTargetException: 'com.sun.tools.javac.tree.JCTree com.sun.tools.javac.tree.JCTree$JCImport.getQualifiedIdentifier()'`

**原因**：spotless 2.29.0 的 `removeUnusedImports` 使用了 JDK 21 已变更的 javac 内部 API。
注意这是 **spotless 自身**的实现问题，仅升级 google-java-format 无效。

**解决**：本分支已升级到 spotless 2.43.0 + google-java-format 1.19.2。

### 5.7 只验证 Java 代码，不想跑前端构建

`seatunnel-web-dist` 包含前端构建（耗时且需下载 node）。
`seatunnel-web-it` 不依赖 `web-dist`，可用它带动全部 Java 模块编译：

```bash
mvn -B -fae clean test-compile -pl seatunnel-web-it -am
```

### 5.8 已知无害告警

构建时会出现以下告警，**不影响产物**（下列计数取自实际打包日志）：

- `datasource-jdbc-db2/pom.xml` 第 66 行重复声明 `seatunnel-api` 依赖
  （`'dependencies.dependency.(groupId:artifactId:type:classifier)' must be unique`）
- `maven-shade-plugin` 报告大量 overlapping resources / class files
  （约 1438 条明细），主要来自 clickhouse、httpclient、avro、jackson 等
  shaded jar 之间的资源与类重复。属第三方依赖打包的常见现象，可忽略。

> 注：日志中出现的
> `Including com.microsoft.sqlserver:mssql-jdbc:jar:6.2.1.jre7 in the shaded jar.`
> 是 INFO 级的打包记录，**不是告警**。

