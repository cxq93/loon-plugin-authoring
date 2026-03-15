---
name: loon-plugin-authoring
description: This skill should be used when the user asks to "写 Loon 插件", "做 Loon 去广告插件", "写 Loon 脚本模板", "配置 Loon Argument 参数", or needs Loon plugin/script authoring and debugging.
version: 0.1.0
---

# Loon 插件编写专用技能（脚本与模板）

本技能聚焦 Loon 插件落地：从插件骨架、参数化配置、脚本模板到验证排障，优先提供可复制、可运行、可回滚的最小方案。

## 使用边界

- 仅处理 Loon 插件编写与脚本联动
- 优先输出插件模板与脚本模板
- 涉及规则分流全局设计时，转交 `loon-ios-manual` 处理总体架构，再回到本技能落地插件细节

## 执行流程

1. 定义插件目标：去广告、重写响应、自动策略切换、定时任务
2. 选择插件最小模块：`[Script]`、`[Rewrite]`、`[MITM]`、`[Argument]`
3. 先生成可用模板，再按目标增量扩展
4. 脚本参数统一走 `argument=[{...}]`，避免硬编码
5. 输出验证步骤、失败信号、回滚动作

## 插件骨架模板

按需给出以下模板并替换占位值：

```ini
#!name= 示例插件
#!desc= 一个可参数化的 Loon 插件模板
#!author= your-name
#!homepage= https://example.com
#!icon= https://example.com/icon.png
#!loon_version= 3.2.1(733)
#!system = iOS,iPadOS,tvOS,macOS
#!tag = 工具,脚本

[Argument]
targetHost = input,"api.example.com",tag=目标域名,desc=脚本匹配域名
enableRewrite = switch,true,tag=启用复写,desc=控制复写规则是否开启
cronExpr = input,"0 */6 * * *",tag=Cron表达式,desc=定时脚本执行周期
mode = select,"safe","strict",tag=运行模式,desc=脚本处理策略

[Script]
http-response ^https?:\/\/.* script-path=https://example.com/script/response.js, requires-body=true, timeout=30, tag=响应处理, enable={enableRewrite}, argument=[{targetHost},{mode}]
cron "{cronExpr}" script-path=https://example.com/script/cron.js, timeout=30, tag=定时任务, argument=[{targetHost},{mode}]

[Rewrite]
^https?:\/\/ads\.example\.com\/.* - reject

[MITM]
hostname = api.example.com,ads.example.com
```

## JavaScript 脚本模板

### A. HTTP 响应脚本模板

```javascript
const args = typeof $argument === "object" && $argument !== null ? $argument : {};
const targetHost = String(args.targetHost || "");
const mode = String(args.mode || "safe");

function done(result) {
  $done(result);
}

try {
  const reqUrl = $request && $request.url ? String($request.url) : "";
  if (targetHost && !reqUrl.includes(targetHost)) {
    done({});
  } else {
    const body = $response && typeof $response.body === "string" ? $response.body : "";
    let nextBody = body;
    if (mode === "strict") {
      nextBody = body.replace(/"ads"\s*:\s*\[[\s\S]*?\]/g, "\"ads\":[]");
    } else {
      nextBody = body.replace(/"ad"/g, "\"_ad\"");
    }
    done({ body: nextBody });
  }
} catch (e) {
  done({});
}
```

### B. Cron 脚本模板

```javascript
const args = typeof $argument === "object" && $argument !== null ? $argument : {};
const targetHost = String(args.targetHost || "");
const mode = String(args.mode || "safe");

const title = "Loon Cron";
const subtitle = "插件定时任务";
const content = `targetHost=${targetHost}, mode=${mode}`;
$notification.post(title, subtitle, content);
$done();
```

## 参数化规范

- 参数入口统一放在 `[Argument]`
- 复写开关只使用 `switch` 类型参数
- 所有脚本都显式声明 `argument=[{...}]`
- 参数默认值必须可安全运行
- 不在脚本内硬编码敏感信息与私有域名

## 常见能力映射

- 去广告：`http-response` + JSON 字段裁剪 + `Rewrite reject`
- Header 改写：`http-request` 脚本修改请求头
- 响应修饰：`http-response` 处理 body，必要时开启 `requires-body=true`
- 自动任务：`cron` 脚本定时通知、状态检测、策略切换
- 策略切换：脚本中调用 `$config` 读取/设置策略组选择

## 验证清单

1. 插件是否能成功导入且元信息显示正常
2. `[Argument]` 控件是否在 UI 正常渲染
3. 脚本是否正确接收 `$argument` 参数
4. `enable={...}` 切换时规则是否按预期开关
5. `cron` 表达式是否可执行且有可观测结果
6. MITM 域名是否完整且证书链可用

## 故障排查

- 脚本不生效：检查 URL 正则、`requires-body`、`timeout`
- 参数未生效：检查 `argument=[{arg}]` 名称与 `[Argument]` 一致性
- cron 不执行：检查 cron 字符串合法性与参数展开结果
- 复写冲突：缩小匹配范围，避免过宽正则
- HTTPS 抓不到：检查 MITM 域名与证书安装状态

## 输出规范

处理用户请求时，固定输出：

1. 场景目标
2. 插件模板
3. 脚本模板
4. 参数说明
5. 验证步骤
6. 回滚方式
