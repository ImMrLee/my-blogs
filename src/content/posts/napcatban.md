---
title: 用Napcat自动化整治Q群熊孩子
published: 2026-03-18
description: ''
image: ''
tags: [Coding]
category: 'Napcat'
draft: false 
lang: 'zh_CN'
---

# 前言
在管理自己的QQ群时，尤其是比较偏低龄化的群聊，很容易会有以下小鬼混入：
- 无脑发各种诡异表情包刷屏的
- 无脑攻击他人的
- 无脑发布各种逆天言论的
一般人遇到这个小屁孩，通常会用禁言，踢出方式处理，但靠禁言的话，你没办法持续且合理地控制，踢出（和永久禁言）的话，一刀切还觉得没意思

------------

所以我选择Napcat全自动化QQ机器人，利用代码之力持续监控熊孩子的发言行为，并按照规则对这个熊孩子进行处罚*~~哈哈哈~~*
## Napcat插件文件结构介绍
参考信息：[https://github.com/NapNeko/napcat-plugin-template](https://github.com/NapNeko/napcat-plugin-template "https://github.com/NapNeko/napcat-plugin-template")
当然，这个仓库还包含插件简单模板，配合文档可以更快上手插件开发
```
napcat-plugin-template/
├── src/
│   ├── index.ts              # 插件入口，导出生命周期函数
│   ├── config.ts             # 配置定义和 WebUI Schema
│   ├── types.ts              # TypeScript 类型定义
│   ├── core/
│   │   └── state.ts          # 全局状态管理单例
│   ├── handlers/
│   │   └── message-handler.ts # 消息处理器（命令解析、CD 冷却、消息工具）
│   ├── services/
│   │   └── api-service.ts    # WebUI API 路由（无认证模式）
│   └── webui/                # React SPA 前端（独立构建）
│       ├── index.html
│       ├── package.json
│       ├── vite.config.ts
│       ├── tailwind.config.js
│       ├── tsconfig.json
│       └── src/
│           ├── App.tsx           # 应用根组件，页面路由
│           ├── main.tsx          # React 入口
│           ├── index.css         # TailwindCSS + 自定义样式
│           ├── types.ts          # 前端类型定义
│           ├── vite-env.d.ts     # Vite 环境声明
│           ├── utils/
│           │   └── api.ts        # API 请求封装（noAuthFetch / authFetch）
│           ├── hooks/
│           │   ├── useStatus.ts  # 状态轮询 Hook
│           │   ├── useTheme.ts   # 主题切换 Hook
│           │   └── useToast.ts   # Toast 通知 Hook
│           ├── components/
│           │   ├── Sidebar.tsx       # 侧边栏导航
│           │   ├── Header.tsx        # 页面头部
│           │   ├── ToastContainer.tsx # Toast 通知容器
│           │   └── icons.tsx         # SVG 图标组件
│           └── pages/
│               ├── StatusPage.tsx  # 仪表盘页面
│               ├── ConfigPage.tsx  # 配置管理页面
│               └── GroupsPage.tsx  # 群管理页面
├── .github/
│   ├── workflows/
│   │   └── release.yml        # CI/CD 自动构建发布
│   ├── prompt/
│   │   ├── default.md             # 默认 Release Note 模板（回退用）
│   │   └── ai-release-note.md     # （可选）AI Release Note 自定义 Prompt
│   └── copilot-instructions.md  # Copilot 上下文说明
├── package.json
├── tsconfig.json
├── vite.config.ts             # Vite 构建配置（含资源复制插件）
```
根据结构图可知，我们的插件目的就是监听，回应消息功能，所以我们只需要处理message-hander.ts就可以了
## 开始写代码
如果第一次编辑message-hander.ts（或者其他文件），会看到里面已经有很多代码内容了，我们可以全部删掉
#### 下面是完整代码示例：
```java
import { pluginState } from '../core/state';

const speakCount: Map<string, number> = new Map();
const targetUid = xxxxxx; // 只针对这个用户

export async function handleMessage(event: any) {
    // 优先用 sender.user_id，如果没有就用 event.user_id
    const uid = event.sender?.user_id ?? event.user_id;
    const gid = event.group_id;

    if (!uid || !gid) {
        console.log("事件对象缺少 uid 或 gid:", event);
        return;
    }
    // 只处理目标用户
    if (uid !== targetUid) {
        return;
    }

    const key = `${gid}-${uid}`;
    const count = (speakCount.get(key) || 0) + 1;
    speakCount.set(key, count);

    console.log(`用户 ${uid} 在群 ${gid} 说话了 ${count} 次`);

    if (count === 10) {
        await pluginState.ctx.actions.call(
            'send_msg',
            {
                message: `用户 ${uid} ，宝宝再说话就要被禁言了哦！`,
                message_type: 'group',
                group_id: gid,
            },
            pluginState.ctx.adapterName,
            pluginState.ctx.pluginManager.config
        );
    }

    if (count > 10) {
        await pluginState.ctx.actions.call(
            'set_group_ban',
            { group_id: gid, user_id: uid, duration: 600 },
            pluginState.ctx.adapterName,
            pluginState.ctx.pluginManager.config
        );
        await pluginState.ctx.actions.call(
            'send_msg',
            {
                message: `用户 ${uid} ，宝宝休息时间到啦！休息10分钟来写写作业吧！`,
                message_type: 'group',
                group_id: gid,
            },
            pluginState.ctx.adapterName,
            pluginState.ctx.pluginManager.config
        );
        speakCount.delete(key); // 重置计数
	}
}
```
这个代码的核心作用就是，统计目标QQ号的人发言次数，若达到10次，则警告一次，若超过10次，就禁言10分钟，并重置计数，特别有意思
