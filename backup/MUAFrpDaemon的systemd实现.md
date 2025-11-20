*作者：UpSGE*

## 前言

由于我们的 frp 进程管理统一使用 `systemd`，网络每天会中断两次，依赖其自动重启等功能，在使用 `python` 实现的 `MUAFrpDaemon` 时出现了诸多问题，且不好统一管理，故写了一个 `systemd` 的实现。

## 通用 systemd 配置

创建 `/etc/systemd/system/frpc@.service` 文件，内容如下：
```ini
[Unit]
Description=Frpc Service: %I
After=network.target

[Service]
User=root
WorkingDirectory=/opt/frp/frpc
ExecStart=/opt/frp/frpc/frpc -c /opt/frp/frpc/%I
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```
这是一个通用的 frpc 启动模板，  
在 `/opt/frp/frpc/` 中放 `frpc` 可执行文件和配置文件，  
即可使用 `systemctl start frpc@example.toml` 命令启动 `frpc`  

**注意**：`%I` 是 `example.toml` 转义后的字符串，可能和原始输入不同导致错误。

## MUAFrp 同步脚本

有了以上通用的 `systemd` 配置，就可以创建一个同步脚本来定时抓取 `MUAFrp` 的配置并启动、调整、终止 `frpc` 进程。

```bash
#!/usr/bin/env bash
# 依赖：bash、curl、jq、systemd

########################
# 配置区
########################

# 临时目录
TMP_DIR="/tmp/union-frpc-tmp"

# 配置目录
TARGET_DIR="/opt/frp/frpc/mua"

# API 配置
MENBER_KEY="xxxxxxxxxxxx"
UNION_API="https://skin.mualliance.ltd/api/union/network"

# metas 配置
DOMAIN="sj.jsumc.fun"
ALIAS=("zj.jsumc.fun" "hb.jsumc.fun")
FORCED_HOSTS=("TEST")

# servers 配置
NAME="TEST"
IP="127.0.0.1"
PORT=60042
REMOTE_PORT=11451

# 日志文件
LOG_FILE="/opt/frp/frpc/muafrp-log.txt"

########################
# 脚本开始
########################

set -u

command -v jq >/dev/null 2>&1 || {
  echo "jq 未安装，请先安装 jq。" >&2
  exit 1
}
command -v curl >/dev/null 2>&1 || {
  echo "curl 未安装，请先安装 curl。" >&2
  exit 1
}

mkdir -p "$TMP_DIR"
mkdir -p "$TARGET_DIR"

# 清理临时目录
rm -f "$TMP_DIR"/* 2>/dev/null || true

# 调用 UNION_API
raw_response=$(curl -sS \
  -H "X-Union-Network-Query-Token: ${MENBER_KEY}" \
  -w '\n%{http_code}' \
  "$UNION_API" || true)

http_code=${raw_response##*$'\n'}
response_body=${raw_response%$'\n'*}

if [ "$http_code" != "200" ]; then
  now="$(date '+%Y-%m-%d %H:%M:%S')"
  {
    echo "[$now]"
    echo "** 请求 $UNION_API 失败，HTTP 状态码: ${http_code:-'N/A'}"
  } >> "$LOG_FILE"
  exit 1
fi

# 构造 metas 字符串
domain_escaped=$(printf '%s' "$DOMAIN" | sed 's/"/\\"/g')
alias_json=$(printf '%s\n' "${ALIAS[@]}" | jq -Rc . | jq -sc .)
alias_escaped=$(printf '%s' "$alias_json" | sed 's/"/\\"/g')
fh_json=$(printf '%s\n' "${FORCED_HOSTS[@]}" | jq -Rc . | jq -sc .)
fh_escaped=$(printf '%s' "$fh_json" | sed 's/"/\\"/g')

metas=$(cat <<EOF
metadatas.domain = "$domain_escaped"
metadatas.domain_alias = "$alias_escaped"
metadatas.forced_hosts = "$fh_escaped"
EOF
)

# 构造 servers 字符串
servers=$(cat <<EOF
[[proxies]]
name = "$NAME"
type = "tcp"
localIp = "$IP"
localPort = $PORT
remotePort = $REMOTE_PORT
EOF
)

# 遍历响应中的 frpc 数组，生成 TMP_DIR 中的配置文件
echo "$response_body" | jq -c '.frpc[]' | while read -r item; do
  id=$(echo "$item" | jq -r '.id')
  config_raw=$(echo "$item" | jq -r '.config')

  # 去掉 Windows 回车
  config=$(printf '%s\n' "$config_raw" | tr -d '\r')

  out_file="$TMP_DIR/$id.toml"
  : > "$out_file"

  # 将 #metas# 和 #servers# 替换为构造内容
  while IFS= read -r line; do
    case "$line" in
      *'#metas#'*)
        printf '%s\n' "$metas" >> "$out_file"
        ;;
      *'#servers#'*)
        printf '%s\n' "$servers" >> "$out_file"
        ;;
      *)
        printf '%s\n' "$line" >> "$out_file"
        ;;
    esac
  done <<< "$config"
done

# 记录增删改的 id
ADDED_IDS=()
UPDATED_IDS=()
DELETED_IDS=()

########################################
# 同步 TMP_DIR -> TARGET_DIR
########################################

# 新增/修改
for tmp_file in "$TMP_DIR"/*; do
  [ -f "$tmp_file" ] || continue
  base_name=$(basename "$tmp_file")
  target_file="$TARGET_DIR/$base_name"

  if [ ! -f "$target_file" ]; then
    # 新增
    cp "$tmp_file" "$target_file"
    systemctl start "frpc@$(systemd-escape "mua/$base_name")" || true
    ADDED_IDS+=("${base_name:0:-5}")
  else
    # 已存在，检查是否有变化
    if ! cmp -s "$tmp_file" "$target_file"; then
      cp "$tmp_file" "$target_file"
      systemctl restart "frpc@$(systemd-escape "mua/$base_name")" || true
      UPDATED_IDS+=("${base_name:0:-5}")
    fi
  fi
done

# 删除多余配置
for target_file in "$TARGET_DIR"/*; do
  [ -f "$target_file" ] || continue
  base_name=$(basename "$target_file")
  if [ ! -f "$TMP_DIR/$base_name" ]; then
    rm -f "$target_file"
    systemctl stop "frpc@$(systemd-escape "mua/$base_name")" || true
    DELETED_IDS+=("${base_name:0:-5}")
  fi
done

########################################
# 写日志
########################################

now="$(date '+%Y-%m-%d %H:%M:%S')"
{
  echo "[$now]"
  if [ "${#ADDED_IDS[@]}" -gt 0 ]; then
    printf '%s' '++'
    for id in "${ADDED_IDS[@]}"; do
      printf ' %s' "$id"
    done
    echo
  fi
  if [ "${#UPDATED_IDS[@]}" -gt 0 ]; then
    printf '%s' '**'
    for id in "${UPDATED_IDS[@]}"; do
      printf ' %s' "$id"
    done
    echo
  fi
  if [ "${#DELETED_IDS[@]}" -gt 0 ]; then
    printf '%s' '--'
    for id in "${DELETED_IDS[@]}"; do
      printf ' %s' "$id"
    done
    echo
  fi
} >> "$LOG_FILE"
```

再使用 `crontab` 定时执行该脚本：

```bash
30 * * * * /opt/frp/frpc/muafrp-sync.sh
```