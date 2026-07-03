# XXL-JOB 本地运行 Runbook（脱敏版）

在这台 Windows 机器上**从零**跑起 `xxl-job7.2`（调度中心 + 执行器）的完整步骤。
所有密码/路径已参数化，照着填自己的值即可复现。

---

## 0. 架构（三个进程）

| 进程 | 端口 | 作用 |
|------|------|------|
| MySQL | 3306 | 调度元数据、任务、日志 |
| xxl-job-admin（调度中心） | 8080 | Web 控制台 + 调度引擎 |
| xxl-job-executor（执行器） | 9999 RPC / 8081 Web | 实际执行任务 |

数据流：调度中心到点 → 路由选一台执行器 → 执行器跑 `@XxlJob` 方法 → 回调结果。

---

## 1. 前提与机器要求

- **JDK 17+**（项目 `source/target=17`；实测 JDK 25 也可）—— 没装的话可用 IDE 自带的 JBR
- **Maven 3.6+**（或用 IDE 自带的）
- **MySQL 8.x**（用已有的，或解压便携版）
- **磁盘**：构建产物 + 依赖 + MySQL 解压 ≈ 1GB+。⚠️ **别放空间紧张的盘**——系统盘满了会让 Windows 页面文件无法扩展，JVM 构建会 OOM
- **内存**：构建峰值约 1GB；运行时三个进程约 1.5GB

---

## 2. 变量约定（先把下面这些在 PowerShell 里赋值，后续命令都引用）

```powershell
$REPO        = "E:\xxl-job7.2"                      # 项目仓库
$BASE        = "E:\run-xxl"                         # 运行环境根目录
$JAVA_HOME   = "E:\IntelliJ IDEA 2026.1.3\jbr"      # JDK（这里借用 IDE 自带）
$MVN         = "E:\IntelliJ IDEA 2026.1.3\plugins\maven\lib\maven3\bin\mvn.cmd"
$DB_PASSWORD = "改成你自己的强密码"                   # MySQL root 密码
$MYSQL_VER   = "8.4.10"
```

---

## 3. 准备目录

```powershell
foreach ($d in @("$BASE","$BASE\mysql","$BASE\mysql-data",
                 "$BASE\executor-logs","$BASE\m2","$BASE\tmp")) {
    New-Item -ItemType Directory -Force -Path $d | Out-Null
}
```

---

## 4. 数据库

### 4.1 下载并解压便携版 MySQL
（已有 MySQL 8.x 的话跳到 4.3，把下面的 `$MYSQL_BIN` 指向你的 bin 目录即可）
```powershell
curl.exe -L -o "$BASE\mysql.zip" "https://cdn.mysql.com/Downloads/MySQL-8.4/mysql-$MYSQL_VER-winx64.zip"
Expand-Archive -Path "$BASE\mysql.zip" -DestinationPath "$BASE\mysql" -Force
$MYSQL_BIN = "$BASE\mysql\mysql-$MYSQL_VER-winx64\bin"
```

### 4.2 写配置文件 `$BASE\my.ini`
```ini
[mysqld]
basedir=<把 $BASE 换成实际路径>/mysql/mysql-8.4.10-winx64
datadir=<把 $BASE 换成实际路径>/mysql-data
port=3306
bind-address=127.0.0.1
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
console
```
> ⚠️ **不要写 `skip-name-resolve`**：加了之后 `root@localhost` 不匹配 TCP 连接，会报 `ERROR 1130 Host '127.0.0.1' is not allowed`。

### 4.3 初始化数据目录 + 启动服务
```powershell
& "$MYSQL_BIN\mysqld.exe" --defaults-file="$BASE\my.ini" --initialize-insecure   # 生成空密码 root@localhost
Start-Process "$MYSQL_BIN\mysqld.exe" "--defaults-file=$BASE\my.ini" -WindowStyle Hidden
& "$MYSQL_BIN\mysqladmin.exe" -u root --password= ping        # 等到返回 "alive"
```

### 4.4 建用户 / 建库 / 导入表结构
凭据用**独立配置文件**传（PowerShell 会弄乱 mysql 客户端的 `-p` 参数）：
```powershell
Set-Content "$BASE\client.cnf" "[client]`r`nhost=127.0.0.1`r`nuser=root`r`npassword=$DB_PASSWORD`r`ndefault-character-set=utf8mb4`r`n"
$mysql = "$MYSQL_BIN\mysql.exe"

& $mysql -u root --password= -e "CREATE USER IF NOT EXISTS 'root'@'127.0.0.1' IDENTIFIED BY '$DB_PASSWORD'; GRANT ALL PRIVILEGES ON *.* TO 'root'@'127.0.0.1' WITH GRANT OPTION; CREATE DATABASE IF NOT EXISTS xxl_job DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 导入建表脚本：cmd 重定向字节干净 + 强制 utf8mb4（否则中文种子数据报 ERROR 1366）
cmd /c "`"$mysql`" --defaults-extra-file=$BASE\client.cnf --default-character-set=utf8mb4 xxl_job < `"$REPO\doc\db\tables_xxl_job.sql`""
```
> 默认管理员：`admin` / `123456`（建表脚本内置，SHA256）。**上线前务必改。**

---

## 5. 构建（命令行参数非常关键，全是踩坑换来的）

### 5.1 Maven 本地仓库重定向到非系统盘
```powershell
Set-Content "$BASE\settings.xml" '<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0">
  <localRepository>E:/run-xxl/m2</localRepository>
</settings>'
```

### 5.2 构建 core + admin + 执行器
```powershell
$env:JAVA_HOME  = $JAVA_HOME
$env:MAVEN_OPTS = "-Xmx512m -XX:MaxMetaspaceSize=256m -XX:ReservedCodeCacheSize=64m -Xss512k -XX:CompressedClassSpaceSize=128m -XX:+UseSerialGC -Djava.io.tmpdir=$BASE\tmp"
$env:PATH       = "$JAVA_HOME\bin;" + $env:PATH
Push-Location $REPO
cmd /c "`"$MVN`" -s $BASE\settings.xml -B clean install -DskipTests -pl xxl-job-core,xxl-job-admin,xxl-job-executor-samples/xxl-job-executor-sample-springboot -am -P !release -Dmaven.compiler.fork=false -Dmaven.compiler.parameters=true > $BASE\build.log 2>&1"
Pop-Location
```

### 5.3 三个不能省的 flag
| Flag | 为什么 |
|------|--------|
| `-P !release` | `release` profile 默认激活，会跑 `maven-gpg-plugin:sign`，没密钥必失败 |
| `-Dmaven.compiler.parameters=true` | `-parameters` 只在 release profile 里；不开的话 Spring 无法按名绑定 `@RequestParam` → **登录报 `IllegalArgumentException ... '-parameters' flag`** |
| `-Dmaven.compiler.fork=false` + 小堆 | 内存/页面文件紧张时，默认堆会让构建 OOM（`Chunk::new`） |

> Maven 输出**直接重定向到文件**（`cmd /c ... > build.log`），别用 PowerShell 管道 `| Tee-Object`——会把 PowerShell 自己撑 OOM。

---

## 6. 启动三个进程

```powershell
# MySQL（已在 4.3 启动则跳过）

# 调度中心 admin（数据源默认指向 127.0.0.1:3306/xxl_job root/<DB_PASSWORD>；
#   若不一致，加 --spring.datasource.url/username/password 覆盖）
Start-Process "$JAVA_HOME\bin\java.exe" "-Xmx384m -XX:MaxMetaspaceSize=200m -jar $REPO\xxl-job-admin\target\xxl-job-admin-3.5.0-SNAPSHOT.jar" -WindowStyle Hidden

# 执行器（日志路径务必覆盖到非系统盘；默认 /data/applogs 会落到 C:）
Start-Process "$JAVA_HOME\bin\java.exe" "-Xmx256m -XX:MaxMetaspaceSize=160m -jar $REPO\xxl-job-executor-samples\xxl-job-executor-sample-springboot\target\xxl-job-executor-sample-springboot-3.5.0-SNAPSHOT.jar --xxl.job.executor.logpath=$BASE\executor-logs" -WindowStyle Hidden
```

- 控制台：**http://127.0.0.1:8080/** → `admin` / `123456`
- 执行器约 30–60s 后自动注册到调度中心（"执行器管理"页能看到地址，IP 是自动探测的）

---

## 7. 日常操作

```powershell
& "$BASE\start-all.ps1"     # 启动全部（幂等，已在跑会跳过）—— 即桌面 START.bat 的内容
& "$BASE\stop-all.ps1"      # 停止全部 —— 即桌面 STOP.bat 的内容
& "$BASE\trigger-job.ps1"   # 手动触发示例任务（job id=1 = demoJobHandler）
```

---

## 8. 常见坑速查

| 现象 | 原因 / 解决 |
|------|------|
| 构建 OOM（`Chunk::new` / native malloc failed） | 页面文件无法扩展（系统盘满）。清磁盘 / 用小堆 / `-Djava.io.tmpdir` 指别的盘 |
| PowerShell 构建 OOM（`format-default OutOfMemoryException`） | Maven 输出别走 PS 管道；用 `cmd /c ... > file 2>&1` |
| 登录 `IllegalArgumentException ... -parameters` | 构建漏了 `-Dmaven.compiler.parameters=true`，重新构建 |
| `ERROR 1130 Host '127.0.0.1' not allowed` | `my.ini` 别加 `skip-name-resolve` |
| 导入 SQL `ERROR 1366 Incorrect string value` | 加 `--default-character-set=utf8mb4` + 用 cmd `<` 重定向 |
| mysql 客户端 `using password: NO` | PS 弄乱了 `-p`；改用 `--defaults-extra-file` 传凭据 |
| 执行器注册不上 | 检查 `appname=xxl-job-executor-sample`、`accessToken=default_token` 与执行器组一致；admin 地址 `http://127.0.0.1:8080` |

---

## 9. `$BASE` 下的脚本/文件清单

| 文件 | 用途 | 含密 |
|------|------|------|
| `my.ini` | MySQL 配置 | 否 |
| `client.cnf` | MySQL 登录凭据（给客户端用） | **是**（DB 密码） |
| `settings.xml` | Maven 本地仓库重定向到 `$BASE\m2` | 否 |
| `setup-db.ps1` | 建用户/建库 | **是**（密码内嵌） |
| `load-schema.ps1` | 导入建表 SQL（utf8mb4） | 否 |
| `build.ps1` / `build-executor.ps1` | 构建 admin+core / 执行器 | 否 |
| `run-admin.ps1` / `run-executor.ps1` | 单独启动 admin / 执行器 | 否 |
| `start-all.ps1`+`START.bat` / `stop-all.ps1`+`STOP.bat` | 一键启停全部 | 否 |
| `verify-registration.ps1` | 检查执行器是否注册到 admin | 否 |
| `trigger-job.ps1` | 登录并触发示例任务 | **是**（admin 口令） |
| `smoke-test.ps1` / `login-test.ps1` | 冒烟/登录验证 | 否 |
| `m2\` | Maven 依赖（179MB，可删后重新下载） | — |
| `mysql\` / `mysql-data\` | MySQL 服务器 + 数据 | — |
| `executor-logs\` | 执行器任务运行日志 | — |

> **提交版本控制前**：`client.cnf`、`setup-db.ps1`、`trigger-job.ps1` 含明文密码，要么不提交、要么先脱敏（密码改用环境变量 `$env:DB_PASSWORD`）。
