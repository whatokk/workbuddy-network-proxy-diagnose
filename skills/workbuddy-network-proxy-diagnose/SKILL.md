---
name: workbuddy-network-proxy-diagnose
description: 诊断并清理 WorkBuddy / 本机的代理阻塞问题。当用户说「联网异常」「代理挡路」「联网慢」「curl exit 7」「删除代理」「子 Agent 502」时使用。覆盖端口归属识别、多路由对比实测、用户级代理变量清理与验证。
agent_created: true
---

# WorkBuddy 网络代理诊断与清理

## 核心认知：分清三类「代理」，别删错

| 类型 | 特征 | 处置 |
|---|---|---|
| WorkBuddy 沙箱代理 | 会话内 `HTTP_PROXY=127.0.0.1:<动态端口>`，端口属于 **sandbox-cli.exe** | **绝不删**，删了沙箱与连接器全断 |
| WorkBuddy 服务代理 | `CODEBUDDY_SERVICE_PROXY_URL=127.0.0.1:<端口>`，端口属于 **WorkBuddy.exe** | 内部机制，不动 |
| 用户级持久代理变量 | `[Environment]::GetEnvironmentVariable(name,'User')` = 某个本机端口 | 这才是通常要清理的对象 |
| FlClash 端口陷阱 | FlClash.exe 监听的口是 **external-controller API 口**，不是代理口 | 别当代理用，会报 getTraffic / BadStatusLine |

## 诊断步骤

### 1. 看会话环境
```bash
env | grep -i proxy
```

### 2. 端口归属（关键，别猜）
```bash
netstat -ano -p tcp | grep -i listening | grep "127.0.0.1:"
tasklist /FI "PID eq <PID>" /FO CSV /NH
```
把 PID 映射到映像名：`sandbox-cli.exe` / `WorkBuddy.exe` / `node.exe`（自建 7890 桥）/ `FlClash.exe`。

### 3. 多路由对比实测（用 Python 比用 curl 更可控）
用 `urllib` + `ProxyHandler({})` 强制直连，构造 5 条路由：直连 / 会话沙箱口 / FlClash 口 / 7890 桥 / 历史死端口，同一 URL 各打一遍，记录 `状态码 + 耗时`。
- 直连 200 但走代理 FAIL → 代理已死，清代理变量。
- 两条都 200 → 网络通道没坏，问题在别处（别乱删）。

### 4. 查持久化配置（Windows）
```powershell
foreach ($n in 'HTTP_PROXY','HTTPS_PROXY','ALL_PROXY','http_proxy','https_proxy','all_proxy') {
  "{0} = {1}" -f $n, [Environment]::GetEnvironmentVariable($n,'User')
}
```
`reg.exe` 被安全策略黑名单拦截，**不要重试、也不要写脚本绕行读注册表**；系统代理用 `netsh winhttp show proxy`。

## 清理动作（可逆，先备份）

```powershell
$names = 'HTTP_PROXY','HTTPS_PROXY','ALL_PROXY','http_proxy','https_proxy','all_proxy'
# 1) 备份
$names | ForEach-Object { "$_=" + [Environment]::GetEnvironmentVariable($_,'User') } | Set-Content <备份路径> -Encoding UTF8
# 2) 删除用户级
$names | ForEach-Object { [Environment]::SetEnvironmentVariable($_, $null, 'User') }
# 3) 读回验证
$names | ForEach-Object { "$_=[" + [Environment]::GetEnvironmentVariable($_,'User') + "]" } | Set-Content <验证路径> -Encoding UTF8
```

- 保留 `NO_PROXY`（旁路清单，不影响）；保留运行中的 7890 桥接进程（子 Agent 通道按会话快照固定连 7890）。
- 环境变量变更**只对新进程生效**，需重启 WorkBuddy / 新开终端。
- 恢复：读备份，用 `SetEnvironmentVariable(name, value, 'User')` 写回。

## 本机踩过的坑

1. **PowerShell 工具 stdout 恒为空**（exit 0 无输出，`Write-Output` 也丢）：命令执行是成功的，只是输出通道哑。绕行＝结果 `Set-Content` 落盘 + Read 读文件验证。
2. **Bash 写 `/tmp/x.py` 会落到 `D:\tmp\`**：Python 找不到文件。脚本一律写到工作区绝对路径。
3. **FlClash 端口漂移**：见过 7890 / 54515 / 54565 / 58595 / 60271。给子 Agent 用的 7890 常驻桥必须留着。
4. 判断「联网异常」不要凭感觉：先跑第 3 步多路由实测，拿数据说话，再决定删不删。
