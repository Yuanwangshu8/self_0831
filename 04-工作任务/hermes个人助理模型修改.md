# Hermes 个人助手：模型修改与微信连通排查

> 状态：已完成  
> 最后更新：2026-09-14  
> 目标：记录 Hermes 连接手机微信时永久更换模型、重启 Gateway、排查无响应和查找宝塔面板的完整流程，便于以后复用。

相关：[[学习hermes]]、[[让obsidian与Hermes联动，实现手机可以书写]]

## 操作截图

![[Pasted image 20260914164953.png]]
![[Pasted image 20260914165130.png]]
![[Pasted image 20260914165316.png]]
![[Pasted image 20260914165434.png]]
![[Pasted image 20260914165653.png]]

## 关键结论

- 永久更换模型使用 `hermes setup`，修改 Base URL、API Key 和模型名后重启 Gateway。
- `hermes -m 模型名` 只对当前一次运行生效，不会改变手机微信使用的模型。
- Hermes 程序目录是 `/usr/local/lib/hermes-agent`，不要直接修改。
- 用户配置目录是 `/root/.hermes`，主配置文件是 `/root/.hermes/config.yaml`。
- API Key 通常在 `/root/.hermes/.env`，不要公开密钥、Token 或宝塔密码。
- 修改配置前先备份，修改完成后必须在微信上实际发消息验证。

## 环境位置

- Hermes 程序目录：`/usr/local/lib/hermes-agent`
- Hermes 用户配置目录：`/root/.hermes`
- 主配置文件：`/root/.hermes/config.yaml`
- 环境变量文件：`/root/.hermes/.env`
- 微信账号配置目录：`/root/.hermes/weixin/accounts/`
- 日志目录：`/root/.hermes/logs/`

## 一、永久更换模型

### 第 1 步：登录服务器

在电脑终端输入：

```bash
ssh root@服务器公网IP
```

成功标志：提示符变成类似 `root@服务器主机名:~#`。

### 第 2 步：备份配置

```bash
cp /root/.hermes/config.yaml /root/.hermes/config.yaml.bak.$(date +%Y%m%d_%H%M%S)
```

成功标志：

```bash
ls -lt /root/.hermes/config.yaml.bak.*
```

可以看到刚生成的备份文件。

### 第 3 步：运行配置向导

```bash
hermes setup
```

向导中的选择：

| 向导内容 | 建议选择 |
| --- | --- |
| Terminal Backend | 保持当前后端 `local`，选择 `Keep current` |
| Messaging Platforms | 微信已正常时，不重新配置微信 |
| Reconfigure Weixin | 选择 `No` |
| Model Provider | 使用原服务时选择 `custom` |
| Base URL | 填写服务商提供的准确地址，通常以 `/v1` 结尾 |
| API Key | 填写新密钥，输入时不要截图或公开 |
| Model | 填写服务商支持的准确模型 ID |
| Restart Gateway | 选择 `Yes` |

成功标志：出现类似：

```text
✓ User service restarted
```

### 第 4 步：验证模型

在服务器终端测试：

```bash
hermes -z "你好，请只回复你当前使用的模型名称"
```

成功标志：终端能够正常返回模型回答。

检查 Gateway 进程：

```bash
pgrep -af 'hermes_cli.main gateway run'
```

成功标志：能够看到类似：

```text
/usr/local/lib/hermes-agent/venv/bin/python -m hermes_cli.main gateway run
```

最后用手机微信发消息测试。微信能够正常回复即表示修改完成。

## 二、腾讯微信无响应时的排查顺序

### 1. 检查 Gateway 是否运行

```bash
pgrep -af 'hermes_cli.main gateway run'
```

- 有进程：继续检查模型和微信连接。
- 无进程：Gateway 没有启动，重新运行 `hermes setup` 并选择重启 Gateway。

### 2. 检查模型是否正常

```bash
hermes -z "你好"
```

如果这里也无法回复，重点检查 API Key、Base URL、模型名、服务商余额、权限和服务器网络。

### 3. 检查错误日志

```bash
tail -n 30 /root/.hermes/logs/errors.log
```

常见错误：

| 错误 | 可能原因 |
| --- | --- |
| `401`、`403` | API Key 错误或服务商权限不足 |
| `404`、`model not found` | 模型名称填写错误 |
| `timeout`、`connection refused` | Base URL 错误或网络不可达 |
| 需要扫码、`weixin disconnected` | 微信连接失效，需要重新配置微信 |

如果微信连接失效，重新运行 `hermes setup`，选择重新配置微信，并用手机微信扫码。

## 三、查找并打开宝塔面板

在服务器终端输入：

```bash
bt default
```

成功标志：输出宝塔面板地址、用户名、密码和安全入口。

宝塔地址通常按下面格式拼接：

```text
https://服务器公网IP:端口/安全入口
```

用电脑浏览器打开该地址。浏览器提示证书风险时，确认地址正确后继续访问。

如果无法打开，检查阿里云安全组和服务器防火墙是否放行宝塔端口。

`systemctl status bt` 显示 `inactive (dead)` 不一定是故障，宝塔有时不以标准 systemd 服务运行。优先使用：

```bash
bt status
```

## 四、注意事项

1. 不要直接修改 `/usr/local/lib/hermes-agent`。
2. 不要公开发送 `/root/.hermes/.env` 的内容。
3. 不要公开发送 `bt default` 输出，因为它包含宝塔账号和密码。
4. 修改模型后一定要重启 Gateway。
5. 配置向导卡在 `Ensuring browser-use CLI` 时，是可选浏览器工具安装阶段，不代表模型配置一定失败。
6. 如果配置被改坏，可以使用之前的 `config.yaml.bak.*` 备份恢复，再重新运行 `hermes setup`。

## 本次结果

已通过 `hermes setup` 完成 Hermes 模型修改和 Gateway 重启，手机微信恢复正常。此任务完成；以后再次更换模型时复用上面的流程。
