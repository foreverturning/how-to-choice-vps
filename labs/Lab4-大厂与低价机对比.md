# Lab 4 大厂轻量与低价 VPS 对比实验

> 做完你会得到：一份同项目、同配置、同时段的三机对比报告；一张能直接用于选型的评分表；一份把首年价、续费价、流量与快照都算进去的账单记录。
> 对应总纲：Lab 4 ｜ 对应文章：[06 选型篇](../chapters/06-选型篇.md) 的 6.3–6.5 ｜ 预计耗时：准备 3–4 小时 + 连续 3 个晚高峰

---

## 目标

把**同一个静态站**分别部署到 A 档头部公有云轻量与 B/C 档低价 VPS，用同一套方法测同一批指标，回答三个问题：

1. 同一份代码，三台机器的首屏、下载、延迟差多少？
2. 差出来的钱，具体对应哪些可测量项——CPU、磁盘、带宽、线路，还是售后与备份？
3. 把续费价摊开之后，哪一档才适合你的场景？

本 Lab 只测单机静态站加一个 1MiB 静态文件，不测数据库、不测高并发、不测 DDoS 清洗。这样做的原因是：变量越少，结论越可信。06 的 6.3 说中小 IDC 的便宜是结构性的，6.4 说海外 VPS 的差价八成在回国线路，6.5 给了十维打分框架；本 Lab 的任务是把这三句话变成你自己机器上的数字。

范围先说清，避免期待错位：

- **能测的**：CPU 单核性能、4K 随机 IOPS、上下行吞吐、延迟与丢包、回程 AS、IPv4 归属、快照能力、工单首响。
- **测不了的**：长期故障率、跑路概率、邻居突然跑满带宽、真实业务并发。这些只能靠时间与口碑，不能靠三天采样。

## 前置条件

| 项目 | 要求 | 说明 |
|---|---|---|
| 三台服务器 | A 档 1 台、B 档 1 台、C 档 1 台 | 统一 2C2G 起；配置不同也能比，但要按单核/每 GB 归一，见常见问题 2 |
| 同一份代码 | 一个静态站目录，含 `index.html`、`style.css`、`1m.bin` | 三台机器校验和必须一致，否则比的不是同一件事 |
| 本地终端 | Linux/macOS shell，或 Windows + WSL | 需要 `ssh`、`scp`、`rsync`、`curl`、`ping`、`mtr` |
| 测速参照机 | 一台与你用户同城的按量实例，或可用的公开 iperf3 节点 | 自建参照机结果更稳，测完释放；公开节点只作参考 |
| 云账号 | 三个独立账号或子账号 | 不要用主账号；不要开自动续费；每个账号都设预算告警 |
| 预算 | 基础组合约 341 元/年，替换 B 档约 611 元/年 | 按快照年价直接相加，不含流量、快照、参照机；会产生费用 |
| 时间 | 准备 3–4 小时 + 连续 3 晚 20:00–22:30 | 采样窗口必须固定，否则数据不可比 |

密钥纪律先立好：私钥只放本地 `~/.ssh/`，权限 600；不写进脚本正文、不进 Git、不发群聊。`.gitignore` 至少加 `*.pem`、`id_ed25519`、`.env`。服务器上只放站点文件与测试脚本，不放你的主账号 AccessKey、支付信息或其他业务密码。

## 步骤

### 步骤 1｜固化同一份被测项目

- **做什么**：在本地建一个 `site/` 目录，放一个最简静态页、一个样式表、一个 1MiB 随机文件。生成后立刻算 SHA-256，存成 `lab4-checksums.txt`。
- **成功标志**：三台机器部署完成后，`sha256sum` 输出与本地记录逐行一致。
- **为什么**：后面所有差异都必须来自机器与线路，不能来自“这台机器上的文件不一样”。

### 步骤 2｜选三台机器并下单（会产生费用）

- **做什么**：按同一档配置选三台，尽量都选 2C2G、40G 左右系统盘、Ubuntu 22.04/24.04。A 档选头部公有云轻量，B 档选中小 IDC，C 档选海外低价 VPS。
- **成功标志**：三台都能 SSH 登录，控制台能看到公网 IPv4、带宽、月流量与计费周期。
- **费用**：A 档 99 元/年（快照 P-003、P-004，2026-11）；B 档 138 元/年（快照 P-011，2026-11）；C 档约 104 元/年（快照 P-018，2026-11）。基础组合合计约 341 元/年。若 B 档改用国内精品线路档 408 元/年（快照 P-013，2026-11，月付 34 元），三台合计约 611 元/年；再加一台 1C1G 海外促销机约 72 元/年（快照 P-017，2026-11），合计约 683 元/年。以上均不含流量超额、快照容量与参照机。
- **下单前截图**：带宽是固定还是峰值、月流量多少、达量是否限速、续费价、退款规则。这五张图后面写报告要用。

### 步骤 3｜三台机器做同一套初始化

- **做什么**：统一时区为 `Asia/Shanghai`，安装同一组工具：`nginx`、`curl`、`mtr-tiny`、`fio`、`sysbench`、`iperf3`、`smartmontools`、`sysstat`、`jq`。不要做内核参数调优，不要装面板，不要开 BBR 之类会引入差异的改动。
- **成功标志**：三台 `nginx -v`、`sysbench --version`、`fio --version` 都能输出版本；`timedatectl` 显示同一时区。
- **记录基线**：发行版、内核、CPU 型号与频率、内存、根分区文件系统，汇总到 `lab4-baseline.txt`，每台一段。海外机若是 Debian 而非 Ubuntu，如实记录，不要强行改成一样。

### 步骤 4｜部署同一份站点，关掉缓存与压缩

- **做什么**：用 `rsync` 把 `site/` 推到三台机器的 `/var/www/lab4/`，套用同一份 Nginx 配置：`gzip off`、`Cache-Control: no-store`、`X-Lab-Host` 响应头。
- **成功标志**：`curl -I http://<IP>/` 返回 200，响应头里有 `Cache-Control: no-store`；三台 `sha256sum` 与本地一致。
- **为什么关缓存与压缩**：压缩率随内容与 CPU 波动，缓存会让第二次请求变成“测本地”，两者都会污染结果。关掉之后，`curl` 每次拿到的都是完整 1MiB。

### 步骤 5｜固定测量口径与时间窗

- **做什么**：把测量分成三层，写进一张 CSV 表头。
  - **一次性指标**（低峰跑）：CPU、4K 磁盘、硬件信息。各跑一次，记录时间。
  - **网络指标**（晚高峰跑）：上行、下行、延迟、丢包、回程。
  - **应用指标**（晚高峰 + 凌晨对照）：HTTP 状态码、TTFB、总耗时、下载速率。
- **成功标志**：`lab4-day1.csv` 表头为 `ts_utc,host,http_code,ttfb_ms,total_ms,speed_kbps,ping_avg_ms,loss_pct`，时间统一 UTC，机器名固定 A/B/C。
- **纪律**：晚高峰定义为北京时间 20:00–22:30，每 15 分钟一轮，共 11 轮；凌晨 03:00 加测一轮作对照。三台机器在同一轮内依次测，顺序固定为 A→B→C；第二天把脚本里的 `for name in A B C` 改成 `C B A`，抵消顺序偏差。

### 步骤 6｜CPU 与 steal 基线（低峰跑）

- **做什么**：`sysbench cpu` 分别跑单线程与双线程各 60 秒，再用 `mpstat` 看 10 秒内的 `%steal`。
- **成功标志**：记录到 `events per second`、`total time`、`%steal` 三个值。
- **判读**：单核 `events/s` 差 2 倍以上，说明 CPU 代际差距明显；`%steal` 长期大于 5%，说明邻居在抢 CPU。注意 `sysbench` 版本不同输出字段略有差异，记录版本号。

### 步骤 7｜磁盘 4K 随机读写（不要在线上机跑）

- **做什么**：先用 `smartctl` 记录硬盘型号、序列号、通电时长或 NVMe 已用寿命；再在 `/tmp` 下建 512MiB 测试文件，`fio` 跑 4K 随机读与随机写，`iodepth=32`、`direct=1`、60 秒、预热 10 秒。
- **成功标志**：记录硬盘型号与通电时长、读/写 IOPS 与 `99.00th`、`99.90th` 延迟。
- **判读**：4K 随机读低于 1,000 IOPS 且 p99 超过 20ms，日常小文件读写会明显卡；NVMe 与 ESSD 的差距通常在这里最直观。写测试只跑一次，跑完删文件。

### 步骤 8｜下载与上传吞吐（会产生流量费用）

- **做什么**：在一台同城按量实例上开 `iperf3 -s`，三台被测机分别向它跑上传与下载，各 30 秒、4 条流、跳过前 3 秒。
- **成功标志**：每台机器拿到两组 `sender/receiver` 速率，重复 3 次取中位数。
- **费用**：30 秒满速的流量约为 `速率(Mbps) × 30 ÷ 8` MB。200M 峰值跑满一轮约 750MB，1Gbps 跑满一轮约 3.75GB。先按标称上限估算，再决定跑几轮；B/C 档的月流量只有 600–1600GB 或 2TB，别把测试跑成超额。
- **判读**：不要拿“200M 峰值”和“3M 固定”直接比数值，先算达成率 = 晚高峰实测 ÷ 标称，再看实际速率。5M 机器的理论上限约 0.625MB/s，跑满 30 秒约 19MB。

### 步骤 9｜延迟、丢包与回程路由

- **做什么**：本地对三台各 `ping` 100 包、`mtr` 100 轮；再从每台服务器向你的本地公网出口做一次 `mtr`，看回程。需要看三网时，用公开 looking glass 分别查电信、联通、移动的回程。
- **成功标志**：记录平均延迟、P95、丢包率、去程与回程经过的 AS。
- **判读**：回程出现 `59.43.x.x` / AS4809 才是 CN2；大量 `202.97` 是 163；`4837` 是联通普通国际；`58453` 是移动 CMI。ICMP 被限速时，改用 `mtr -T -P 443` 走 TCP。

### 步骤 10｜应用层响应与 IPv4 归属

- **做什么**：本地对三台各发 5 次 `curl`，每次带时间戳查询串破缓存，记录状态码、DNS、连接、TTFB、总耗时、下载速率。再在服务器上比对 `curl -4 ifconfig.me` 与 `ip -4 addr`。
- **成功标志**：每台 5 行数据；出口 IP 与网卡 IP 一致或不一致，有明确结论。
- **判读**：出口 IP 与 `ip addr` 不一致，说明是 NAT 或共享出口；`whois` 的 `country`、`netname` 与商家宣称地域不符，记一笔。`whois` 可能被限速，失败就换 RDAP 网页查询。

### 步骤 11｜连续 3 天晚高峰采样（会产生流量费用）

- **做什么**：用同一支脚本，在每天 20:00 启动，跑 11 轮、每轮间隔 15 分钟，输出 `lab4-day1.csv`、`lab4-day2.csv`、`lab4-day3.csv`。
- **成功标志**：每个文件 33 行数据（11 轮 × 3 台），无空值；三天的 `ts_utc` 落在同一时间段。
- **费用**：1MiB × 11 轮 × 3 台 × 3 天 ≈ 99MiB，约 0.1GB，可忽略；真正的大头是步骤 8 的 iperf3。若把测试文件换成 10MiB，流量乘以 10。
- **判读**：看中位数与 P95，不看单次峰值；同时统计“失败轮次”，这就是可用性的最粗口径。

### 步骤 12｜账单、续费价、快照/备份、工单与汇总评分（可能产生快照费用）

- **做什么**：五项一起收口。
  1. **账单**：截图首年价、实付、计费周期、本月流量、超额费、快照容量费。
  2. **续费价**：进控制台续费页询价，记录“续费同价”“限 1 次”“活动至某年某月”等条款。A 档促销档的续费价约 600–720 元/年（快照 P-001、P-002，2026-11，待核对），必须按续费价重算，不能拿 99 元当长期成本。
  3. **快照/备份**：是否有快照、是否免费、单价多少（待核对）、创建与回滚各花多久。没有快照的直接记“不提供”。
  4. **工单**：用同一句话在工作时间提交，例如“请问如何查看本月已用流量？”，记录首响时间与解决时间；不要发账号、密钥、身份证等信息。
  5. **评分**：按 6 维权重填分，价格维一律用续费价 ÷ 本档年地板（A 600 元、B 180 元、C 36 元）。首年价会让 A 档直接变成黑灯，这是首年补贴的正常现象，不是机器有问题。
- **成功标志**：五张记录齐全，总分能复算。

## 命令

以下命令可直接复制。先把三个 IP 与密钥路径改成你自己的值；密钥只引用路径，不要把私钥内容贴进脚本。

```bash
export A_IP="<A 档服务器IP>"
export B_IP="<B 档服务器IP>"
export C_IP="<C 档服务器IP>"
export SSH_KEY="$HOME/.ssh/lab4_ed25519"
export REF_IP="<同城参照机IP>"
```

**步骤 1：生成同一份被测项目**

```bash
mkdir -p site && cd site
cat > index.html <<'EOF'
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Lab 4 对比站</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <h1>Lab 4 对比站</h1>
  <p>同一份代码，部署在三台机器上。</p>
  <p><a href="/1m.bin">下载 1MiB 测试文件</a></p>
</body>
</html>
EOF
printf 'body{font-family:system-ui,sans-serif;margin:4rem;line-height:1.6}\n' > style.css
head -c 1048576 /dev/urandom > 1m.bin
sha256sum index.html style.css 1m.bin | tee ../lab4-checksums.txt
cd ..
```

**步骤 3：三机初始化（三台执行同一段）**

```bash
for ip in "$A_IP" "$B_IP" "$C_IP"; do
  ssh -i "$SSH_KEY" root@"$ip" 'bash -s' <<'EOF'
set -e
export DEBIAN_FRONTEND=noninteractive
apt update
apt install -y nginx curl mtr-tiny fio sysbench iperf3 smartmontools sysstat jq whois
timedatectl set-timezone Asia/Shanghai
systemctl enable --now nginx
EOF
done
```

**步骤 3：记录硬件与系统基线**

```bash
: > lab4-baseline.txt
for name in A B C; do
  eval ip=\$${name}_IP
  {
    echo "===== $name ====="
    ssh -i "$SSH_KEY" root@"$ip" '
      hostname
      date -u
      grep -E "^(PRETTY_NAME|VERSION_ID)=" /etc/os-release
      uname -r
      lscpu | grep -E "Model name|^CPU\(s\)|Thread|Core|MHz"
      free -m
      df -hT /
    '
  } >> lab4-baseline.txt
done
```

**步骤 4：部署站点与统一 Nginx 配置**

```bash
cat > lab4.conf <<'EOF'
server {
    listen 80 default_server;
    server_name _;
    root /var/www/lab4;
    index index.html;
    gzip off;
    add_header Cache-Control "no-store" always;
    add_header X-Lab-Host $hostname always;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

for ip in "$A_IP" "$B_IP" "$C_IP"; do
  rsync -az --delete -e "ssh -i $SSH_KEY" ./site/ root@"$ip":/tmp/lab4-site/
  scp -i "$SSH_KEY" lab4.conf root@"$ip":/etc/nginx/sites-available/lab4.conf
  ssh -i "$SSH_KEY" root@"$ip" '
    mkdir -p /var/www/lab4
    rsync -a --delete /tmp/lab4-site/ /var/www/lab4/
    chown -R www-data:www-data /var/www/lab4
    ln -sf /etc/nginx/sites-available/lab4.conf /etc/nginx/sites-enabled/lab4.conf
    rm -f /etc/nginx/sites-enabled/default
    nginx -t && systemctl reload nginx
  '
done
```

**步骤 4：校验三台文件一致**

```bash
for ip in "$A_IP" "$B_IP" "$C_IP"; do
  echo "== $ip =="
  curl -s -o /dev/null -w "%{http_code} %{size_download}\n" "http://$ip/1m.bin"
  ssh -i "$SSH_KEY" root@"$ip" "sha256sum /var/www/lab4/index.html /var/www/lab4/1m.bin"
done
```

**步骤 6：CPU 与 steal**

```bash
for name in A B C; do
  eval ip=\$${name}_IP
  echo "===== $name ====="
  ssh -i "$SSH_KEY" root@"$ip" '
    sysbench cpu --cpu-max-prime=20000 --threads=1 --time=60 run | grep -E "events per second|total time"
    sysbench cpu --cpu-max-prime=20000 --threads=2 --time=60 run | grep -E "events per second|total time"
    mpstat 1 10 | tail -n 3
  ' | tee "lab4-cpu-$name.txt"
done
```

**步骤 7：4K 随机读写（只在测试机跑，跑完删文件）**

```bash
for name in A B C; do
  eval ip=\$${name}_IP
  echo "===== $name ====="
  ssh -i "$SSH_KEY" root@"$ip" '
    lsblk -d -o NAME,MODEL,ROTA,SIZE
    for d in /dev/sda /dev/vda /dev/nvme0n1; do
      [ -b "$d" ] && smartctl -a "$d" | grep -E "Model|Serial|Power_On_Hours|Percentage Used|Data Units Written"
    done
    mkdir -p /tmp/fio-lab4 && cd /tmp/fio-lab4
    fio --name=randread4k --ioengine=libaio --direct=1 --rw=randread --bs=4k \
        --iodepth=32 --numjobs=1 --size=512M --runtime=60 --time_based \
        --ramp_time=10 --group_reporting | grep -E "read: IOPS|99.00th|99.90th"
    fio --name=randwrite4k --ioengine=libaio --direct=1 --rw=randwrite --bs=4k \
        --iodepth=32 --numjobs=1 --size=512M --runtime=60 --time_based \
        --ramp_time=10 --group_reporting | grep -E "write: IOPS|99.00th|99.90th"
    rm -rf /tmp/fio-lab4
  ' | tee "lab4-fio-$name.txt"
done
```

**步骤 8：上下行吞吐（先开参照机，测完关掉）**

```bash
# 参照机终端 1：启动服务端
ssh -i "$SSH_KEY" root@"$REF_IP" "iperf3 -s -p 5201"

# 被测机终端 2：上传与下载各 3 次
for run in 1 2 3; do
  for name in A B C; do
    eval ip=\$${name}_IP
    echo "===== run $run / $name 上传 ====="
    ssh -i "$SSH_KEY" root@"$ip" "iperf3 -c $REF_IP -p 5201 -t 30 -P 4 -O 3"
    echo "===== run $run / $name 下载 ====="
    ssh -i "$SSH_KEY" root@"$ip" "iperf3 -c $REF_IP -p 5201 -t 30 -P 4 -O 3 -R"
  done
done
```

**步骤 9：延迟、丢包与回程**

```bash
for name in A B C; do
  eval ip=\$${name}_IP
  echo "===== $name ====="
  ping -c 100 -i 0.2 -q "$ip" | tail -n 2
  mtr -rwzbc 100 "$ip" | tail -n 15
done | tee "lab4-network-$(date +%F).txt"
```

```bash
# 回程：在服务器上测回你的本地出口
export MY_IP="<你的本地公网出口IP>"
for name in A B C; do
  eval ip=\$${name}_IP
  echo "===== $name 回程 ====="
  ssh -i "$SSH_KEY" root@"$ip" "mtr -rwzbc 50 $MY_IP | tail -n 15"
done
```

```bash
# 去程路由与 AS（TCP 探测需要 root 权限，macOS/Linux 用 sudo）
for ip in "$A_IP" "$B_IP" "$C_IP"; do
  echo "== $ip =="
  sudo traceroute -T -p 443 "$ip"
done
```

**步骤 10：应用层响应与 IPv4 归属**

```bash
for name in A B C; do
  eval ip=\$${name}_IP
  for i in 1 2 3 4 5; do
    curl -o /dev/null -s \
      -w "$name,%{http_code},%{time_namelookup},%{time_connect},%{time_starttransfer},%{time_total},%{speed_download}\n" \
      "http://$ip/1m.bin?t=$(date +%s%N)"
    sleep 1
  done
done | tee "lab4-curl-$(date +%F).csv"
```

```bash
for name in A B C; do
  eval ip=\$${name}_IP
  echo "===== $name $ip ====="
  ssh -i "$SSH_KEY" root@"$ip" "curl -4 -s https://ifconfig.me; echo; ip -4 addr show scope global | grep inet"
done
```

```bash
for ip in "$A_IP" "$B_IP" "$C_IP"; do
  echo "== $ip =="
  whois "$ip" | grep -Ei "^(country|netname|org-name|descr):" | head -n 5
done
```

**步骤 11：连续 3 天晚高峰采样**

```bash
cat > lab4-sample.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
DAY=${1:?用法: bash lab4-sample.sh day1}
OUT="lab4-${DAY}.csv"
echo "ts_utc,host,http_code,ttfb_ms,total_ms,speed_kbps,ping_avg_ms,loss_pct" > "$OUT"
for round in $(seq 1 11); do
  ts=$(date -u +%FT%TZ)
  for name in A B C; do
    eval ip=\$${name}_IP
    if ! read -r code ttfb total speed < <(curl -o /dev/null -s \
      -w "%{http_code} %{time_starttransfer} %{time_total} %{speed_download}" \
      "http://$ip/1m.bin?t=$(date +%s%N)"); then
      code=000; ttfb=0; total=0; speed=0
    fi
    ping -c 20 -i 0.2 -q "$ip" > /tmp/lab4-ping.out || true
    avg=$(awk -F'/' '/rtt|round-trip/ {print $5}' /tmp/lab4-ping.out)
    loss=$(awk '/packet loss/ {for(i=1;i<=NF;i++) if($i=="packet") print $(i-1)}' /tmp/lab4-ping.out | tr -d '%')
    echo "$ts,$name,$code,$(awk -v v="$ttfb" 'BEGIN{printf "%.1f", v*1000}'),$(awk -v v="$total" 'BEGIN{printf "%.1f", v*1000}'),$(awk -v v="$speed" 'BEGIN{printf "%.1f", v*8/1000}'),${avg:-0},${loss:-100}" >> "$OUT"
  done
  if [ "$round" -lt 11 ]; then sleep 900; fi
done
EOF
bash lab4-sample.sh day1
```

**步骤 11：三天数据汇总（均值、最小值、最大值、中位数）**

```bash
awk -F, 'FNR>1 {k=$2; n[k]++; s[k]+=$7; if($7>mx[k])mx[k]=$7; if(mn[k]==""||$7<mn[k])mn[k]=$7}
END {for (k in n) printf "%s n=%d avg=%.1f min=%.1f max=%.1f\n", k, n[k], s[k]/n[k], mn[k], mx[k]}' \
  lab4-day1.csv lab4-day2.csv lab4-day3.csv
```

```bash
for name in A B C; do
  echo -n "$name median="
  awk -F, -v h="$name" 'FNR>1 && $2==h {print $7}' lab4-day*.csv \
    | sort -n \
    | awk '{a[NR]=$1} END{print (NR%2)?a[(NR+1)/2]:(a[NR/2]+a[NR/2+1])/2}'
done
```

> 上面两段只用 POSIX awk 与 `sort`，不依赖 gawk 扩展；如果数字对不上，先把三份 CSV 合并进表格软件复核，别为了脚本折腾半小时。

## 产出

做完本 Lab，你手上应该有五样东西：

1. **三机部署记录**：三台机器的 IP 代号、系统版本、CPU 型号、内存、磁盘、带宽与月流量，外加三份一致的 SHA-256。
2. **原始数据**：`lab4-cpu-*.txt`、`lab4-fio-*.txt`、`lab4-network-*.txt`、`lab4-curl-*.csv`、`lab4-day1..3.csv`。原始文件不要改，分析另存。
3. **一张对比表**：按下表填实测值，空着的格子就是你没测的项。

| 维度 | A 档（头部公有云轻量） | B 档（中小 IDC） | C 档（海外低价 VPS） | 结论（怎么判） |
|---|---|---|---|---|
| 年价（首年 / 续费） | 99 / 99（活动期或限 1 次） | 138 / 138 | 约 104 / 待核对 | 一律用续费价算 R |
| CPU 单核（events/s） | ____ | ____ | ____ | 差 2 倍以上说明代际差 |
| 4K 随机读 IOPS / p99 | ____ | ____ | ____ | <1,000 且 p99 >20ms 偏差 |
| 4K 随机写 IOPS | ____ | ____ | ____ | 看是否被限 IO |
| 晚高峰下载（Mbps） | ____ | ____ | ____ | 除以标称得达成率 |
| 晚高峰上传（Mbps） | ____ | ____ | ____ | 回国方向常不对称 |
| 延迟 avg / P95（ms） | ____ | ____ | ____ | 必须同城参照 |
| 丢包（%） | ____ | ____ | ____ | 晚高峰 >3% 记异常 |
| 回程 AS / 线路 | ____ | ____ | ____ | 59.43 / 202.97 / 4837 |
| IPv4 归属 | ____ | ____ | ____ | 与宣称地域一致？ |
| 快照 / 备份 | 有，按量 | 多为另购或无 | 多为无 | 无快照 = 数据风险 |
| 工单首响（分钟） | ____ | ____ | ____ | 同为工作时间口径 |
| 3 天可用性 | ____ | ____ | ____ | 失败轮次 ÷ 总轮次 |

4. **一张评分表**：按下表加权，分数必须来自上面的实测值，不能凭印象。

| 维度 | 权重 | 打分依据 | A | B | C |
|---|---:|---|---:|---:|---:|
| 价格 vs 成本地板 | 30 | 续费价 ÷ 本档年地板（600 / 180 / 36） | | | |
| 资源真实性 | 20 | 单核、4K IOPS、steal、内存 | | | |
| 网络质量 | 15 | 晚高峰达成率、丢包、回程 | | | |
| 稳定 / SLA | 15 | 3 天可用性、快照、故障 | | | |
| 售后 / 合规 | 10 | 工单首响、发票、备案 | | | |
| 隐性限制 | 10 | 达量限速、流量、续费条款 | | | |
| **合计** | **100** | 🟢 ≥80 / 🟡 60–79 / 🔴 40–59 / ⚫ <40 | | | |

5. **一张账单表**：首年价、续费价、流量、快照、参照机各一行，标注快照编号或“待核对”。可直接并入 [Lab 5 预算告警与复盘](./Lab5-预算告警与复盘.md) 的成本表。

验收自检，五条全过才算完成：

- [ ] 三台 `sha256sum` 与 `lab4-checksums.txt` 完全一致
- [ ] CPU、4K、上下行、延迟丢包、回程、IPv4 六类原始数据都有文件
- [ ] 三天晚高峰 CSV 各 33 行，时间戳落在 20:00–22:30
- [ ] 续费价已经查到，不是拿首年价当长期成本
- [ ] 评分表每一项都能指回一条原始数据

## 常见问题

**1. 测速时段不一致，三台数据对不上**

晚高峰的带宽是共享的，20:00 和 23:00 能差一倍。固定窗口（20:00–22:30）、固定轮次（11 轮）、固定顺序（A→B→C，下一轮反向），并统一用 UTC 时间戳记录。若某天错过窗口，宁可整晚作废，也不要把 23:30 的数据混进 20:00 的序列。

**2. 三台机器配置不同，还能比吗？**

能比，但要先归一。CPU 看单核 `events/s`，不看总核数；内存按每 GB 折算；磁盘看 IOPS 与 p99，不看容量；网络先算“实测 ÷ 标称”的达成率，再比数值。配置差异必须在报告里写明，不能假装三台一样。如果一台是 1C1G、另一台是 2C2G，结论只能写“这个价位段的表现”，不能写“某档不如某档”。

**3. 缓存干扰，第二次测特别快**

三层缓存都要破：浏览器/系统缓存、Nginx 的 `gzip` 与 `expires`、CDN 边缘节点。本 Lab 用 `gzip off` + `Cache-Control: no-store` + 查询串破缓存；如果你套了 CDN，测试期间先关掉代理。`curl` 每次新连接，别用长连接复用。看到“第一轮慢、后面飞快”，先怀疑缓存，不要急着夸机器。

**4. 线路波动，今天好明天差**

单次 `iperf3` 没有意义。至少 3 次取中位数，跨 3 天看 P95；晚高峰丢包从 0% 跳到 8% 是常事，尤其是普通国际线路。记录时把“测速时间、方向、并发数、是否绕路”一起写，否则一周后你自己也解释不了这个数。若三天里有一天异常，保留原始数据并在报告里标注，不要删掉。

**5. 账单口径不同，看不出谁贵**

四个口径先统一：首年价还是续费价、年付还是月付折年、含不含带宽与流量、含不含 IP 与快照。A 档 99 元/年可能只限一次续费同价（快照 P-004，2026-11），B 档 138 元/年写着续费同价（快照 P-011，2026-11），C 档 104 元/年是促销价、续费待核对（快照 P-018，2026-11）。报告里一律用“续费价 ÷ 12”写月成本，把一次性费用单列。

**6. iperf3 跑不满，是机器差还是测法错？**

先查三件事：安全组/防火墙是否放了 5201、是不是单线程被 TCP 窗口限制、参照机自己够不够快。用 `-P 4` 多流、`-O 3` 跳过慢启动、`-R` 测反方向。5M 机器跑出 5Mbps 就是满速，不要拿它和 200M 峰值比数值。若 `iperf3` 正常而 `curl` 很慢，问题在应用层或线路，不在带宽。

## 安全提示

1. **测速会消耗流量**。iperf3 跑满 200M 峰值 30 秒约 750MB，跑满 1Gbps 约 3.75GB；B/C 档月流量只有 600–1600GB 或 2TB，先算再跑。三天采样约 0.1GB，可以忽略，但别把测试文件换成 100MiB。
2. **别在生产机跑 fio**。4K 随机写会打满磁盘 IO、消耗 SSD 寿命，还可能触发商家的 IO 限额。只在测试机、`/tmp` 下跑，`--size=512M` 封顶，跑完删文件。线上机要看磁盘，用 `iostat` 只读观察。
3. **别把密钥写进脚本**。脚本里只引用 `$SSH_KEY` 路径，不贴私钥内容；不把 `lab4.conf`、CSV、账单截图里的 IP 与账号信息提交到公开仓库。`.gitignore` 加 `*.pem`、`id_*`、`.env`、`lab4-*.csv`。
4. **别用主账号**。三个实验账号或子账号，单独设密码、单独开预算告警、关闭自动续费。主账号绑着支付方式与备案主体，一旦泄露，损失不止三台机器。
5. **测完释放资源**。按量参照机当天释放；临时安全组规则（5201）测完删掉；测试快照按需保留，不用的删；年付机到期前决定续不续，别让“忘了取消”变成第二年的账单。
6. **只测自己的机器**。不要对第三方站点跑 iperf3、mtr 压测或端口扫描；不要用测试流量去打别人的 IP。工单里只问计费与功能，不发密钥、身份证、银行卡信息。

## 费用清单

| 项目 | 量级 | 标注 | 说明 |
|---|---|---|---|
| A 档轻量 2C2G | 99 元/年 | （快照 P-003、P-004，2026-11） | 3–4M；续费同价限 1 次或活动期内 |
| B 档 2C2G（香港） | 138 元/年 | （快照 P-011，2026-11） | 200M 峰值共享，600–1600GB/月 |
| B 档 2C2G（国内精品 BGP） | 408 元/年（34 元/月） | （快照 P-013，2026-11） | 5M 宣称独享，需验真 BGP |
| C 档 2C2G（海外） | 约 104 元/年 | （快照 P-018，2026-11） | 1Gbps 共享，2TB/月，无中国优化 |
| C 档 1C1G（海外促销） | 约 72 元/年 | （快照 P-017，2026-11） | 年末促销，非长期价 |
| 同城参照机（按量） | 几元–十几元 | （待核对） | 测完立即释放 |
| 快照 | 每月几元量级 | （待核对） | 按占用容量计费，部分套餐含额度 |

读法：这张表只回答“一共大概多少钱”。基础组合 99 + 138 + 104 ≈ 341 元/年；把 B 档换成 408 元档，约 611 元/年；再加一台 72 元档，约 683 元/年。区间与“待核对”项不做精确加总，流量超额与快照另计。

## 什么场景选哪档

把上面的数据折成一张落地表。分数只在你自己的场景里有效，别拿别人的报告直接下单。

| 场景 | 首选 | 次选 | 不建议 | 理由 |
|---|---|---|---|---|
| 静态作品集 / 文档站 | 免费静态托管（快照 P-022） | A 档最低配 | C 档年付 | CDN 免费且快，服务器是多余的一层 |
| 小工具（日活 <100，要后端） | A 档轻量 2C2G | B 档月付 | C 档年付 | 备案、工单、快照三项都要人兜底 |
| 国内商用（备案 + 发票） | A 档 | — | B / C 档 | 合规是硬门槛，低价档给不了 |
| 出海展示（用户在国外） | C 档普通线路 | A 档境外轻量 | CN2 溢价款 | 用户不在国内，回国优化是白花钱 |
| 学习 / 实验 | A 档促销或 B 档月付 | C 档 | 多年年付 | 随时可能弃用，别把风险提前买断 |
| 有出网流量的图片 / 下载站 | 对象存储 + CDN | A 档按流量 | B/C 的“不限量” | 达量限速与超量计费才是大头 |

读法：先看“用户在哪”和“要不要备案发票”两列，再看价格。A 档赢在合规、售后与快照，B 档赢在月付灵活与线路选择，C 档赢在便宜与免备案；三者的便宜与贵，都能在上面的实测表里找到对应项。

## 完成后你应该得到什么

- 一份**同条件对比报告**：同一份代码、同一套初始化、同一个时间窗，三台机器的 CPU、磁盘、上下行、延迟丢包、回程、IPv4、快照、工单各有数字。
- 一张**能复算的评分表**：6 维权重、续费价口径、三色灯阈值与 05、06 完全一致，换一台机器也能套。
- 一份**真实账单记录**：首年价、续费价、流量、快照、参照机各一行，知道钱花在哪，也知道第二年大概要花多少。
- 一个**可迁移的方法**：以后看到任何“几十元一年”的报价，你会先问配置、再问续费、再跑三项测试，而不是先付款。
- 一个**明确的取舍**：知道自己当前的项目该选 A、B 还是 C，以及为什么另外两档不合适。

如果这一步走通了，下一步可以拿 B/C 档机器去做 [Lab 6 低价 VPS 体检](./Lab6-低价VPS体检.md)，把硬件来源、IP 信誉、合规资质补上；预算与告警部分接到 [Lab 5 预算告警与复盘](./Lab5-预算告警与复盘.md)。选型框架本身见 [06 选型篇](../chapters/06-选型篇.md) 的 6.5，成本地板与三色灯见 [05 成本解剖篇](../chapters/05-成本解剖篇.md)。

## 免责声明

> 本文价格为 2026-11 的公开信息快照与量级估算，会随厂商调价、汇率、地区与配置变化而不同；文中不对低价机、转卖机与中小商家点名，引用头部公有云公开定价或财报时仅作出处，不构成购买推荐或收益承诺。实际下单前请以厂商官网、合同与工单答复为准。
