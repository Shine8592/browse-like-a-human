# browse-like-a-human · 像人一样浏览

**一个让浏览器自动化通过机器人检测的 Agent 技能——靠「本来就是」，不是靠「绕过去」。**

[English](README.md) · [技能定义](SKILL.md) · [参考文档](references/)

---

## 先说清楚这个区别

让浏览器通过 Cloudflare Turnstile、hCaptcha 或任何「验证你是人」的关卡，有两条路。

**绕过**，是打败这个检查：从打码平台取一个 token、复用别人拿到的 clearance cookie、patch 掉检测脚本、花钱买人机验证。这条路在厂商更新检测规则之后就失效——通常几周之内。而且在平台眼里，这**就是**一次攻击。每一次绕过，都会在日志里留下"发生过绕过"的痕迹。

**拟人浏览**，是让检测没有东西可检测。真实浏览器内核、完整指纹、操作系统级的真实输入事件、干净的住宅出口。平台的风控引擎看到的是一次正常会话——因为它检查的每一个信号都是**真的**，不是被伪装成真的。

```
绕过：伪造信号，让检测"以为"你是人
      → 检测一升级 → 你失效
      → 他们在日志里看到的是一段异常

拟人：在一切可观测维度上，你就是人
      → 检测再升级 → 什么都不变
      → 你与中位数用户无法区分
```

这个仓库讲的是第二条路。它更难搭建，但维护起来容易得多。

---

## 自动化浏览器到底死在哪一层

所有「我的 Playwright 脚本被封了」的案例，最后都能归到同一个三层结构上。排查要从外往内、逐层排除——而绝大多数人一直在调最里面那层。

| 层 | 平台检查什么 | 典型失败表现 | 修复难度 |
|---|---|---|---|
| **① IP 信誉** | ASN、机房 vs 家宽、proxy/VPN 标记、滥用历史、地理一致性 | AWS/Oracle/DigitalOcean 的 IP，被见过一万次 | 容易——换出口 |
| **② 行为** | 鼠标熵、击键时序、滚动物理、焦点切换、请求节奏 | `element.click()` 让光标瞬移；`fill()` 一毫秒写入 500 字符 | 中等——注入真实事件 |
| **③ 指纹** | `navigator.*`、WebGL、canvas、字体、音频、TLS/JA3、客户端提示一致性 | `brands: null`、`webdriver: true`、WebGL 渲染器 `ERR` | 中等——补上泄漏点 |

**最省时间的一条判断**：如果页面从头到尾**没有渲染出 Turnstile 组件**，或者每次刷新 Ray ID 都在变、永远停在过渡页——**别再改浏览器参数了**。第一层已经挂了，第二、三层根本没机会被评估。

```js
// 诊断：是有东西可点，还是在你点到之前就已经被拒了？
document.querySelectorAll('iframe').length   // 0 → 纯 JS 挑战，IP 层被拒
document.querySelector('[name="cf-turnstile-response"]')?.value.length
// 自建浏览器下为 0   → 指纹或出口不对（可修）
// 云沙箱里为 0       → 这个页面本来就没要求你过验证
```

---

## 决定成败的三处指纹

在真实的两级 Turnstile 流程上实测：三项全绿 + 住宅出口，**两道全部无人干预通过**——第一道还是 invisible 形式，勾选框根本没出现过。

| 信号 | 错误值 | 正确值 |
|---|---|---|
| `navigator.userAgentData.brands` | `null` | `[{Google Chrome, 131}, {Chromium, 131}, {Not_A Brand, 24}]` |
| WebGL `UNMASKED_RENDERER_WEBGL` | `ERR` | `ANGLE (Intel, Intel(R) UHD Graphics 630 Direct3D11 ...)` |
| `navigator.webdriver` | `true` | `undefined` |

三项里 `brands: null` 最致命。**没有任何一个已发布的 Chrome 会在这返回 null**——这等于直接认罪。

而造成第二项的那个参数，在无头/Xvfb 环境里几乎人人都在用，就是 `--disable-gpu`。它看起来很合理，实际上毁掉了你的 WebGL 渲染器。用软件光栅化替代：

```bash
# 错误
--disable-gpu
# 正确
--use-gl=angle --use-angle=swiftshader --enable-unsafe-swiftshader
```

注入方法、一致性校验的细节见 [`references/fingerprint-hardening.md`](references/fingerprint-hardening.md)

---

## 鼠标问题：XSendEvent 与 XTEST

这是整个仓库里价值最高的一条经验。

在 X11 上移动光标有两种机制：

| 机制 | 实际做了什么 | 风控看到什么 |
|---|---|---|
| `XSendEvent`（多数"桌面自动化"工具） | 合成一个事件投递给窗口 | 一个合成事件——窗口从未真正被点过 |
| `XTEST`（`xdotool`、XInput2） | 在设备/驱动层注入，早于窗口系统 | 一个真实的指针在移动，一次真实的按键 |

用 `XSendEvent`，可以把 Turnstile 的勾选框像素级精确命中，然后**它一动不动**。这不是"失败"，是**完全没有反应**——在风控看来，这比答错更可疑。

换成 `XTEST`，同一个坐标会触发 `Verifying you are human…` 的转圈，然后通过。

```bash
# cua-driver 这类工具 → XSendEvent → 被无视
# xdotool → XTEST → 被接受
xdotool mousemove --sync 812 447
sleep 0.1
xdotool click 1
```

再叠上人类运动学特征——缓入缓出的轨迹、25–28 步、±1.5 px 抖动、步间隔 8–22 ms、按下前先随机游走一小段。完整参数和可运行代码见 [`references/human-mouse.md`](references/human-mouse.md)

> **绝不在同一个流程里混用两种机制。** 二十次真实点击里混进一次合成事件，就足以让整个会话失败——而且它会制造一个看起来像"网站有问题"的 bug，实际上是你自己不统一。

---

## 目录结构

```
SKILL.md                        技能本体——触发条件、操作流程、红线
references/
  diagnosis.md                  三层模型、探针、决策树
  fingerprint-hardening.md      三处泄漏、注入、一致性校验
  human-mouse.md                XTEST vs XSendEvent、运动学、可运行代码
  turnstile-mechanics.md        invisible / managed / interactive、令牌判据
  operations.md                 虚拟桌面搭建、UI 点击纪律、进程坑
  verification.md               真客户端端到端验证
```

---

## 快速开始

```bash
# 1. 虚拟桌面
Xvfb :99 -screen 0 1440x900x24 -nolisten tcp &
DISPLAY=:99 openbox &            # 缺窗口管理器会让所有点击静默失效

# 2. 住宅出口走独立接口——绝不动你的生产隧道
#    （单独起一个 tunN + 独立路由表 + 独立代理端口）

# 3. 指纹修好的真实浏览器会话，挂上代理
#    → 见 references/fingerprint-hardening.md
```

然后用真实客户端验证，不要靠猜——见 [`references/verification.md`](references/verification.md)。手动拼 WebSocket 升级头去 `curl` 代理服务器，**一切正常时也会返回 400**；`400` 说明服务活着，`502/503` 才是真的挂了。

---

## 边界（说清楚）

这个技能是给**以你自己身份做的合法自动化**用的：

- 创建和管理你本就有权拥有的账号
- 测试你自己系统的反机器人防御
- 研究检测机制到底怎么运作

明确**不**用于：

- 打码平台、令牌转售、任何第三方 token 服务
- 批量注册或一次性垃圾账号
- 绕过你不拥有的系统的限速或访问控制
- 任何违反目标站点服务条款的行为

这里的技术让浏览器与普通用户无法区分。**这是一个属性，不是一个漏洞**——而且和其他所有能力一样，它的正当性取决于你拿它做什么。如果某个平台的条款禁止自动化注册，那么"技术上能做到"并不等于"这样做没问题"。

仓库内不含任何凭据、令牌、个人数据或针对具体站点的绕过方案。所有内容都做泛化描述，以便它能扛过下一次检测更新。

---

## 许可

MIT — 见 [LICENSE](LICENSE)

---

## 来源

全部内容来自实战笔记：在真实 Cloudflare 防护的注册流程上反复摸索，每一次失败都记下了它的根因。**失败的记录才是价值所在**——成功的那几次，主要只是没有犯本可避免的错误而已。
