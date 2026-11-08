# TG 自动签到 · Muvyo 第三方插件仓库

一个可以直接发布的 Muvyo 插件仓库：**5 条独立的签到任务**，每条自己填「名称 / 聊天对象 / 聊天内容」，在 Muvyo 的「定时任务」里给每条设自己的时刻，到点把内容发给对应的 Telegram 机器人。

> 先说明平台规则，避免走弯路：Muvyo 的定时任务**必须在打包时写进 `manifest.json` 的能力声明里**，运行时不能新增任务（上限 8 个）。所以「最多 5 条、每条可自定义」的实现方式是：清单里预置 5 个任务槽 `t1–t5`，每个槽配 3 个配置项。任务数量要改，只能改清单 + 重新签名发版。
>
> 同理，能发消息的机器人也必须写在清单的 `telegram_bots` 里（最多 5 个）。这是 Muvyo 的安全设计：安装时用户能看到插件会给哪些机器人发消息。

---

## 一、仓库长什么样

```
muvyo-tg-checkin/                    ← 你的 GitHub 仓库（公开）
  .gitignore                         忽略 src/、签名凭据、私钥
  .githooks/pre-commit               提交检查：拦住源码与私钥
  .claude/skills/muvyo-addon-publish/scripts/sign_index.py
                                     官方模板的仓库索引签名工具
  src/sbet123.tg-checkin/           源码（不提交）
    manifest.json
    main.js
    README.md
  index.json                         签名后生成（提交）
  index.json.sig                     仓库索引签名（提交）
  plugins/sbet123.tg-checkin/       签名后的产物（提交）
```

发布只提交 `index.json`、`index.json.sig`、`plugins/`。`src/` 不上传。

## 二、一次性准备

1. **Muvyo 永久授权** + 在 Muvyo「插件 → 第三方插件 → 开发者」点「用 GitHub 账号领取证书」，按提示在 GitHub 输入验证码。
2. 下载得到 `muvyo-signing.json`（作者证书 + 私钥，**绝不能提交**）和 `mv_addon.py`（签名脚本），都放到本仓库根目录。
3. 本机装 Python 3 与依赖：

   ```powershell
   pip install cryptography
   ```

4. 在 GitHub 新建一个 **public** 仓库（例如 `muvyo-tg-checkin`），把这套文件推上去（或先在本地建好再关联远端）。

## 三、已经填好的内容（不用再改）

- 插件 ID：`sbet123.tg-checkin`（前缀 = 你的 GitHub 用户名 `sbet123`）
- 作者：`sbet123`
- 机器人白名单：`yingshiziyuan_bot`
- 5 条任务的「聊天对象」已预填 `yingshiziyuan_bot`，安装后不用再填

要加别的签到机器人：在 `permissions.telegram_bots` 数组里追加（**不带 @**、以 `bot` 结尾、最多 5 个），升 `version` 后重新签名发布。没写进这里的机器人，插件调用时会被拒绝——这就是「聊天对象」的白名单。

### 可选：改显示名、描述、任务标题

`name`、`description`、`tasks[].title` 随你改；`id` 定了就别再动（改了就是另一个插件）。

## 四、签名并发布

### 1. 签插件

```powershell
python mv_addon.py sign .\src\sbet123.tg-checkin -o .
```

产物落到根目录：`index.json`、`plugins/sbet123.tg-checkin/{manifest.json, encrypted.json, README.md, signature.json}`。检查一下：

```powershell
python mv_addon.py verify .\plugins\sbet123.tg-checkin
```

### 2. 签仓库索引（首次要建仓库专用密钥）

```powershell
# 只做一次：生成仓库签名私钥（会写进 .gitignore，务必另行备份）
python .claude\skills\muvyo-addon-publish\scripts\sign_index.py init-key --key repo-signing.pem

# 每次发布
python .claude\skills\muvyo-addon-publish\scripts\sign_index.py sign --key repo-signing.pem
python .claude\skills\muvyo-addon-publish\scripts\sign_index.py verify
```

`sign` 会把公钥写进 `index.json` 并生成 `index.json.sig`。**索引任何改动都要重新 sign**，否则 Muvyo 拒绝读取仓库。

### 3. 推送

```powershell
git config core.hooksPath .githooks     # 启用提交检查，防止误传 src/ 和私钥
git add index.json index.json.sig plugins
# 首次提交还要加上仓库骨架：
# git add README.md .gitignore .githooks .claude
git commit -m "发布 TG 自动签到 1.0.0"
git push
```

发布纪律：只用默认加密签名（不要 `--no-encrypt`），不上传 `src/`、明文 JS 或任何凭据。提交检查会拦下这些文件。

## 五、在 Muvyo 里安装和使用

1. 「第三方插件 → 插件仓库」添加仓库地址：`https://github.com/sbet123/muvyo-tg-checkin`。
2. 列表里点安装，确认联网范围：**使用 Muvyo 的 Telegram 机器人**（会列出 `yingshiziyuan_bot`）、无额外网站域名。
3. 插件设置里填 **任务 1–5**：
   - 名称：随便写，用于区分；
   - 聊天对象：已预填 `yingshiziyuan_bot`，不用动；
   - 聊天内容：例如 `/checkin`（1–500 字，最多 4 行，具体命令以机器人提示为准）。
4. 打开「定时任务」，给每个「每日签到 N」设定执行时刻（签到类默认是每天固定时刻）。用不到的任务把聊天内容清空，插件会跳过；如果界面能单独关闭某个任务，建议直接关掉，免得每天多出几条「跳过」记录。
5. 到点后结果进「签到」通知；设置页顶部会显示每条任务最后一次的成功/失败。

> 同一插件同一时间只执行一个任务，5 条尽量错开几分钟；每个插件每天最多发 30 条，5 条/天完全够用。

## 六、以后改内容 / 加任务

- **只改配置**（名称、机器人、内容、时间）：直接在 Muvyo 设置里改，不用重新签名。
- **换机器人**（不在清单里的）：改 `manifest.json` 的 `telegram_bots` → 升 `version` → 重新签名 → 重新签索引 → 推送。
- **改成 3 条或 8 条**：改 `capabilities.tasks`（上限 8）、对应增删 `config` 字段与 `settings.sections`，再升版本签名发布。
- **更新**：用户端会看到「可更新」，确认后生效。

## 七、常见问题

**安装提示「插件 ID 必须以作者命名空间开头」**：`id` 前缀和 GitHub 用户名（小写）不一致。

**提示「插件未获准访问 …」**：本插件不需要任何网站域名，`permissions.domains` 保持 `[]` 即可。如果以后要直连 `api.telegram.org`，必须写进 `domains` 并在安装时确认。

**任务报「机器人不在…」**：机器人没写进 `telegram_bots`，或写的时候带了 `@`/大小写不符。改清单重新签名。

**任务没执行**：检查「定时任务」里这个任务是否打开、时刻是否已过、Muvyo 的 Telegram 账号是否在线。

**提示「今天发送次数已用完」**：平台限制每插件每天 30 条，5 条任务不会碰到；如果反复手动测试才会用完。

**能不能发给群 / 频道 / 任意聊天对象**：不能。`telegram.bot` 服务只能给清单里声明的**机器人**发文字。要发群/频道，得改成用你自己的 Bot Token 直连 `api.telegram.org`（声明该域名 + 用 `secret` 配置存 token），但那条路走的是你自己的机器人，不是「用你的账号给签到机器人发命令」，签到场景不适用。

**私钥丢了怎么办**：Muvyo 开发者页「作废这张证书」后重新领取、重新签名；仓库私钥丢了就无法再签索引，只能换新仓库地址（Muvyo 首次添加时记住公钥）。

## 许可

示例结构参考 [thsrite/muvyo-addon-guide](https://github.com/thsrite/muvyo-addon-guide)（模板 MIT-0）。本插件代码随你使用。
