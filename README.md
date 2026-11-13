# 星河 & 无界签到 · Muvyo 第三方插件仓库

这个仓库发布两个 Muvyo 第三方签到插件，都是「每天给一个 Telegram 机器人发一条命令，把回复整理成签到通知」：

| 插件 | 插件 ID | 命令 | 机器人 | 版本 |
|---|---|---|---|---|
| 星河 | `sbet123.xinghe` | `/sign` | @yingshiziyuan_bot | 1.0.3 |
| 无界签到 | `sbet123.wujie` | `/checkin` | @boundlesscloud_bot | 1.0.3 |

## 功能

- 定时任务「每日签到」：默认每天一次，由 Muvyo 的「定时任务」面板调度。
- 发送各自命令，等机器人回复，按机器人实际字段整理成通知：
  - 星河：获得积分、连续签到天数、当前积分；
  - 无界签到：余额 / 流量奖励、下次可签到时间；当天已签到则显示距下次签到倒计时（回复里有积分字段时也会解析）。
- 返回给 Muvyo 的消息就是签到通知正文，例如：`获得积分 5，连续签到 6 天，当前积分 282`、`余额 +0.03 元，下次可签到 2026-05-09 00:52`；缺哪项就不显示哪项。
- 插件设置页顶部显示上次签到的时间、积分和结果。
- 机器人没回复时只标「结果未知」，不重试，避免重复签到。

## Muvyo 里的任务面板

安装后打开「定时任务」，每个插件各有一个 **每日签到**，自带这些选项（和癫影插件一致，都是 Muvyo 平台提供的）：

| 选项 | 说明 |
|---|---|
| 自动运行 | 打开后才会按计划执行 |
| 推送签到结果 | 把上面的结果推到「签到」通知 |
| 执行方式 | 每天固定时刻 / 间隔 / 时段内随机 |
| 随机时段 | 例如 08:00–10:00，按 Muvyo 的随机时段执行 |

无界签到的默认间隔是 **24 小时 1 分钟**：第一次按你在「定时任务」里设定的时刻执行，之后每次比上次晚 24 小时 1 分钟（比机器人 24 小时冷却多 1 分钟，避免卡边界回「已签到」）。星河仍是每天一次。

## 仓库结构

```
muvyo-addons/                        ← GitHub 仓库（公开）
  .gitignore                         忽略 src/、签名凭据、私钥
  .gitattributes                     签名产物按原始字节存储（见下）
  .githooks/pre-commit               提交检查：拦住源码与私钥
  .claude/skills/muvyo-addon-publish/scripts/sign_index.py
                                     仓库索引签名工具
  src/sbet123.xinghe/                星河源码（不提交）
  src/sbet123.wujie/                 无界签到源码（不提交）
  index.json                         签名产物（提交）
  index.json.sig                     仓库索引签名（提交）
  plugins/sbet123.xinghe/            星河签名产物（提交）
  plugins/sbet123.wujie/            无界签到签名产物（提交）
```

发布只提交 `index.json`、`index.json.sig`、`plugins/`。`src/` 不上传。

## 发布步骤（换机器时照做）

1. 装 Python 3 与 `cryptography`：`pip install cryptography`。
2. 在 Muvyo「插件 → 第三方插件 → 开发者」用 GitHub 账号领证书，把 `muvyo-signing.json` 和 `mv_addon.py` 放到本仓库根目录（两者都已被忽略）。
3. 签名插件（改了哪个就签哪个）：

   ```powershell
   python mv_addon.py sign .\src\sbet123.xinghe -o .
   python mv_addon.py sign .\src\sbet123.wujie -o .
   python mv_addon.py verify .\plugins\sbet123.xinghe
   python mv_addon.py verify .\plugins\sbet123.wujie
   ```

4. 签索引（首次建 `repo-signing.pem`，之后只跑 sign + verify）：

   ```powershell
   python .claude\skills\muvyo-addon-publish\scripts\sign_index.py init-key --key repo-signing.pem
   python .claude\skills\muvyo-addon-publish\scripts\sign_index.py sign --key repo-signing.pem
   python .claude\skills\muvyo-addon-publish\scripts\sign_index.py verify
   ```

5. 提交推送：

   ```powershell
   git add -A
   git commit -m "发布 星河 / 无界签到"
   git push
   ```

## 安装

1. Muvyo「第三方插件 → 插件仓库」添加：`https://github.com/sbet123/muvyo-addons`。
2. 安装「星河」，确认使用 Telegram 机器人 `yingshiziyuan_bot`；再安装「无界签到」，确认 `boundlesscloud_bot`。
3. 各自插件设置里确认「签到命令」：星河 `/sign`，无界签到 `/checkin`。
4. 分别打开「定时任务 → 每日签到」，打开自动运行，按需打开推送签到结果，选择执行方式和时段。
5. 两个插件建议错开几分钟，避免同一时间排队执行。

## 发布时的换行坑（重要）

签名哈希是对**文件原始字节**算的。Windows 上签名脚本写出的 `index.json`、`manifest.json`、`README.md` 是 CRLF，而 Git 默认会把它们转成 LF，线上字节和签名对不上，Muvyo 会报「插件文件与签名不符」。

`.gitattributes` 里已经处理：

```
index.json -text
index.json.sig -text
plugins/** -text
```

这几条让 Git 原样存字节，**不要删**。每次推送后对 GitHub 上取到的文件验一次签更稳妥。

另外 GitHub raw 有约 5 分钟 CDN 缓存：刚推送完就点更新，可能读到一半新一半旧的缓存，报 README 不一致。等 5 分钟再点即可。

## 更新版本

1. 改对应 `src/sbet123.xxx/` 里的代码或清单，`manifest.json` 的 `version` 升号。
2. 重新签名该插件和索引（上面的第 3、4 步）。
3. 提交推送；用户在插件仓库里会看到「可更新」。

两个插件的版本号互相独立，只改一个不会影响另一个。

## 常见问题

**任务没执行**：确认「定时任务」里对应插件的「每日签到」自动运行已打开、时刻已到、Muvyo 的 Telegram 账号在线。

**通知里没有积分**：说明机器人回复里没有可识别的字段，通知会直接显示机器人原文；把原文发我，我可以加解析规则。

**提示「插件未获准使用 Telegram 机器人」**：安装确认时没允许，重装或更新时勾选。

**提示「插件文件与签名不符」**：先等 5 分钟排除 raw 缓存；仍不行就检查 `.gitattributes` 是否还在，重新签名推送。

**能不能发群/频道**：不能，`telegram.bot` 只能给清单里声明的机器人发文字。

## 许可

结构参考 [thsrite/muvyo-addon-guide](https://github.com/thsrite/muvyo-addon-guide)（模板 MIT-0）。
