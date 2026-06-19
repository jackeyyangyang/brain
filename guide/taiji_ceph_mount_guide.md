# 太极平台 Ceph 个人盘挂载实录与解决方案

> 适用客户端：`taiji_client`（Version ≥ 2.1.0）
> 场景：在非标准（社区版 ceph-fuse）环境下，挂载太极平台提供的个人 private ceph 盘
> 验证时间：2026-06-19

## 关键信息（请妥善保管）

| 项 | 值 |
|----|----|
| 存储挂载 token | `qSI7UqgZAqOjCX7ePbMZlg` |
| 账号 | `jhyangyang` |
| 推荐城市 | 重庆 `cq`（深圳 `sz` 平台侧建目录失败，暂不可用） |
| 默认挂载点 | `/apdcephfs/private_jhyangyang` |

> 安全提示：token 等同于访问凭证，请勿将本文件公开或提交到代码仓库。

---

## 零、一键挂载脚本（全新环境推荐，复制即用）

> 适用于「换了一个临时 GPU 环境、需要重新挂载同一块个人盘」的场景。
> 脚本已包含全部前置检查与处理，**正常情况下复制粘贴执行一次即可挂载成功**。

```bash
#!/bin/bash
set -e
TOKEN="qSI7UqgZAqOjCX7ePbMZlg"
CITY="cq"                                  # 推荐重庆；深圳 sz 暂不可用
MOUNT_POINT="/apdcephfs/private_jhyangyang"

echo "===> [1/5] 检查 taiji_client 是否安装"
if ! command -v taiji_client >/dev/null 2>&1; then
  echo "未找到 taiji_client，请先按平台文档安装后重试。"; exit 1
fi
taiji_client --version || true

echo "===> [2/5] 确保社区版 ceph-fuse 已安装（wrapper 依赖它）"
if [ ! -x /usr/bin/ceph-fuse ]; then
  echo "未发现 /usr/bin/ceph-fuse，尝试用 yum 安装社区版 ..."
  yum install -y ceph-fuse || { echo "ceph-fuse 安装失败，请手动安装"; exit 1; }
fi
/usr/bin/ceph-fuse --version || true

echo "===> [3/5] 创建 ceph-fuse wrapper（过滤腾讯定制选项）"
cat > /usr/local/bin/ceph-fuse <<'EOF'
#!/bin/bash
# ceph-fuse wrapper：过滤掉社区版不支持的腾讯定制选项，再转发给真正的 ceph-fuse
args=()
for a in "$@"; do
  case "$a" in
    --fuse_fake_tmp_agent*|--client_not_support_security*|--client_setuid_optimize*)
      ;;  # 丢弃上游 ceph-fuse 不识别的定制选项
    *)
      args+=("$a") ;;
  esac
done
exec /usr/bin/ceph-fuse "${args[@]}"
EOF
chmod +x /usr/local/bin/ceph-fuse
hash -r
echo "当前 ceph-fuse 指向：$(which ceph-fuse)"   # 期望 /usr/local/bin/ceph-fuse

echo "===> [4/5] 创建挂载点并执行挂载"
mkdir -p "$MOUNT_POINT"
taiji_client mount -l "$CITY" -tk "$TOKEN"

echo "===> [5/5] 验证挂载结果"
mount | grep -q "$MOUNT_POINT" && echo "挂载成功：$MOUNT_POINT" || { echo "挂载未生效，请看下方排障"; exit 1; }
df -h "$MOUNT_POINT"
touch "$MOUNT_POINT/.write_test" && echo "写入成功" && rm -f "$MOUNT_POINT/.write_test" && echo "删除成功"
```

> 说明：
> - 若新环境**自带 dop-fuse**（PATH 中有 `dop-fuse`），`taiji_client mount` 会原生成功，wrapper 不会被用到，也不影响。
> - 若 `which ceph-fuse` 仍指向 `/usr/bin/ceph-fuse`，说明 `/usr/local/bin` 不在 PATH 前面，需执行 `export PATH=/usr/local/bin:$PATH` 后重试。

---

## 一、最终结果

个人 ceph 盘已成功挂载并可正常读写：

| 项目 | 值 |
|------|-----|
| 挂载点 | `/apdcephfs/private_jhyangyang` |
| 城市 | 重庆（cq） |
| 容量 | 2.0T（已用 0%） |
| 文件系统类型 | `fuse.ceph-fuse` |
| 状态 | 可正常读写 |

卸载命令：

```bash
umount /apdcephfs/private_jhyangyang
```

---

## 二、问题背景与现象

直接执行官方文档命令挂载个人盘时，三个城市表现不同：

```bash
taiji_client mount -l sz -tk <token>   # 深圳
taiji_client mount -l cq -tk <token>   # 重庆
taiji_client mount -l qy -tk <token>   # 清远
```

| 城市 | 现象 | 失败阶段 |
|------|------|----------|
| 深圳 sz | `[error][-1100605][mkdir /user/jhyangyang fail]` | **服务端 API 查询阶段** |
| 重庆 cq | 服务端查询成功，但本地 `fuse failed to start` | **本地挂载阶段** |
| 清远 qy | 同重庆，本地 `fuse failed to start` | **本地挂载阶段** |

---

## 三、根因定位

### 1. 深圳（sz）—— 平台侧问题，客户端无法解决

通过 `strace` 抓取 HTTP 请求/响应链路：

- 客户端向 `taijiapi.oa.com` 发起请求：
  - 接口：`POST /taskmanagement/task_server/config_management/api/query/private_ceph_storage/`
  - 方法：`EXT_STORAGE_QUERY`，参数 `{"location":"sz"}`
- 服务端返回：`{"code": -1100605, "message": "mkdir /user/jhyangyang fail"}`

即：**太极平台在深圳 ceph 集群侧为该账号创建私有目录 `/user/jhyangyang` 失败**。
该错误发生在 API 查询阶段（早于任何本地挂载动作），因此客户端侧任何改动（自定义挂载点、wrapper、sudo 等）都无法绕过，**需要太极平台运维修复账号在深圳集群的目录创建问题**。

### 2. 重庆/清远（cq/qy）—— 本地 ceph-fuse 版本不兼容

服务端查询正常并返回了完整挂载参数（monitor 地址、keyring、client 名、远程路径），但本地挂载报错：

```
fuse: unknown option(s): `--fuse_fake_tmp_agent=true --client_not_support_security=true --client_setuid_optimize=true'
ceph-fuse[xxxx]: fuse failed to start
```

根因：

- `taiji_client` 默认通过 **dop-fuse** 挂载；当前环境 PATH 中无 dop-fuse，回退到标准 `ceph-fuse`。
- 通过 `strace -e trace=execve` 追踪到它实际调用 `/usr/bin/ceph-fuse`，并强制传入 3 个**腾讯定制版（ceph-fuse3）专属选项**：
  - `--fuse_fake_tmp_agent`
  - `--client_not_support_security`
  - `--client_setuid_optimize`
- 但当前环境装的是 **社区上游版 `ceph-fuse 18.2.0`**，不认识这些定制选项 → `fuse failed to start`。
- 排查 `yum`：源里只有上游社区版 ceph-fuse，**没有腾讯定制的 ceph-fuse3，也没有独立的 dop-fuse 包**（现有 dop-fuse 挂载是容器启动时平台注入的，无可重用二进制）。

---

## 四、解决方案（核心成功经验）

由于无法安装腾讯定制版 ceph-fuse3，采用 **ceph-fuse 包装脚本（wrapper）** 的方式：在 PATH 中优先于 `/usr/bin` 的 `/usr/local/bin` 放一个 `ceph-fuse`，自动过滤掉上游不支持的 3 个定制选项，再转发给真正的 `/usr/bin/ceph-fuse`。这样 `taiji_client` 的完整挂载流程（写 keyring → 调用 ceph-fuse → 挂载）即可跑通。

### 步骤 1：创建 wrapper 脚本

创建 `/usr/local/bin/ceph-fuse`：

```bash
#!/bin/bash
# ceph-fuse wrapper：过滤掉上游社区版不支持的腾讯定制选项，
# 再转发给真正的 /usr/bin/ceph-fuse
args=()
for a in "$@"; do
  case "$a" in
    --fuse_fake_tmp_agent*|--client_not_support_security*|--client_setuid_optimize*)
      # 丢弃这些上游 ceph-fuse 不识别的定制选项
      ;;
    *)
      args+=("$a")
      ;;
  esac
done
exec /usr/bin/ceph-fuse "${args[@]}"
```

### 步骤 2：赋权并刷新 PATH 缓存

```bash
chmod +x /usr/local/bin/ceph-fuse
hash -r
which ceph-fuse        # 应输出 /usr/local/bin/ceph-fuse
```

> 确认 `/usr/local/bin` 在 PATH 中优先于 `/usr/bin`（通常默认如此）。

### 步骤 3：挂载（选用服务端正常的城市，如重庆 cq）

```bash
mkdir -p /apdcephfs/private_jhyangyang
taiji_client mount -l cq -tk qSI7UqgZAqOjCX7ePbMZlg
```

### 步骤 4：验证

```bash
mount | grep private_jhyangyang
df -h /apdcephfs/private_jhyangyang
ls -la /apdcephfs/private_jhyangyang/
# 读写测试
touch /apdcephfs/private_jhyangyang/.write_test && echo "写入成功" \
  && rm -f /apdcephfs/private_jhyangyang/.write_test && echo "删除成功"
```

---

## 五、排查方法论（可复用的调试手段）

| 手段 | 命令示例 | 作用 |
|------|----------|------|
| 抓网络链路 | `strace -f -e trace=network taiji_client mount ...` | 找出客户端连接的服务端地址 |
| 抓 HTTP 内容 | `strace -f -e trace=read,write,sendto,recvfrom -s 4096 ...` | 看服务端返回的 JSON 错误码/消息 |
| 抓实际执行 | `strace -f -e trace=execve -s 300 ...` | 看它实际 exec 哪个 fuse 二进制及参数 |
| 检查 fuse 版本 | `ceph-fuse --version` | 区分社区版 / 腾讯定制版 |
| 检查可装包 | `yum list available \| grep -iE "ceph-fuse\|dop"` | 确认能否安装定制版 |
| 检查现有挂载 | `cat /proc/mounts \| grep -i dop` | 判断 dop-fuse 是否平台注入 |

---

## 六、注意事项

1. **wrapper 为本会话临时方案**：容器/环境重建后会失效，重新执行「第零节一键脚本」或「第四节步骤 1~3」即可。
2. **深圳（sz）暂不可用**：服务端建目录失败属平台侧问题，需提工单给太极运维修复账号在深圳集群的目录。修复前请使用重庆（cq）或清远（qy）。
3. **持久化建议**：如需开机/登录自动生效，可将 wrapper 创建逻辑写入启动脚本（如 `~/.bashrc` 或容器初始化脚本）。
4. **token 处理**：开发机首次填写 `-tk` 后本地会保存，后续可省略；太极任务环境无法本地保存，每次需带 `-tk`。
5. **数据持久性**：写入挂载目录（`/apdcephfs/private_jhyangyang/`）的数据存放在远程 ceph 集群，**不会因临时 GPU 环境注销而丢失**；丢失的只是本地挂载映射和 wrapper 脚本，重新挂载即可访问同一份数据。请勿把重要数据放在 `/root`、`/tmp` 等本地路径。

---

## 七、换新环境一次性挂载 Checklist

在全新临时环境中，按下表确认即可一次挂载成功（第零节脚本已自动覆盖这些点）：

| # | 检查项 | 命令 | 不满足时的处理 |
|---|--------|------|----------------|
| 1 | taiji_client 已安装 | `command -v taiji_client` | 按平台文档安装 |
| 2 | 社区版 ceph-fuse 已安装 | `/usr/bin/ceph-fuse --version` | `yum install -y ceph-fuse` |
| 3 | wrapper 已就位且优先 | `which ceph-fuse` | 重建 wrapper；`export PATH=/usr/local/bin:$PATH` |
| 4 | 挂载点已创建 | `ls -d /apdcephfs` | `mkdir -p /apdcephfs/private_jhyangyang` |
| 5 | 选用可用城市 | —— | 用 `cq`，勿用 `sz` |
| 6 | 执行挂载并验证 | `taiji_client mount -l cq -tk qSI7UqgZAqOjCX7ePbMZlg` | 见下方排障 |

### 常见报错对照排障

| 报错 | 原因 | 解决 |
|------|------|------|
| `fuse: unknown option(s): --fuse_fake_tmp_agent ...` | 走了社区版 ceph-fuse 且未经 wrapper | 创建/修复 wrapper（第零节步骤 3），`hash -r` 后重试 |
| `fuse failed to start` | 同上 | 同上 |
| `[error][-1100605][mkdir /user/jhyangyang fail]` | 深圳平台侧建目录失败 | 改用 `-l cq`；深圳需提工单 |
| `/usr/bin/ceph-fuse: No such file` | 社区版 ceph-fuse 未安装 | `yum install -y ceph-fuse` |
| `which ceph-fuse` 指向 `/usr/bin` | `/usr/local/bin` 不在 PATH 前 | `export PATH=/usr/local/bin:$PATH` |

---

## 八、常用命令速查

```bash
# 挂载（重庆，推荐）
taiji_client mount -l cq -tk qSI7UqgZAqOjCX7ePbMZlg

# 挂载到自定义路径
taiji_client mount -l cq -tk qSI7UqgZAqOjCX7ePbMZlg /mnt/private

# 回退使用 ceph-fuse（本环境默认即走此路径）
taiji_client mount -l cq -tk qSI7UqgZAqOjCX7ePbMZlg --use_ceph

# 卸载
umount /apdcephfs/private_jhyangyang

# 升级客户端
taiji_client update
```
