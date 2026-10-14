# 💧 Liquid Glass Web — 苹果液态玻璃的 Web 实现

纯 HTML / CSS / JavaScript 实现的 **Apple Liquid Glass（iOS 26 液态玻璃）**，包含真实折射、果冻物理、体积光与互相照亮。零依赖、单文件即可运行。

> 折射引擎逐字移植自 [shuding/liquid-glass](https://github.com/shuding/liquid-glass)（MIT），在其基础上扩展了完整的交互动效层。

## ✨ 效果一览

| 效果 | 说明 |
|---|---|
| 🫧 **液态折射** | `backdrop-filter: url(#svg)` + 内缩 SDF 位移贴图，中心平坦、边缘一圈强烈弯曲，附 RGB 边缘色散 |
| 🫳 **果冻按压** | 按下 `scale(1.05, 0.93)` 压扁下陷 + 底部内阴影；**按住拖动**时按钮朝鼠标方向【局部】鼓出去（`transform-origin` 反侧偏移技巧），松手 500ms 弹簧回弹 |
| 💡 **体积光** | 光晕完全贴合按钮轮廓（外扩 14% + blur 22px），整个按钮柔和发光，而非点光源 |
| 🌟 **光照互通** | 每块玻璃内置受光层，受光面朝向光源、距离线性衰减——按钮按下会照亮附近的玻璃，玻璃拖过也会互相照亮 |
| 🌊 **流体拖拽** | 指数平滑速度采样、体积守恒拉伸（沿任意速度方向）、惯性滑行（摩擦 0.94/帧）、停止余晃、快甩发光 |
| ⏱️ **弹簧动画** | `springEase(t) = 1 − e^(−5t)·cos(9t)`，500ms 一个过冲周期，页面内有实时曲线可视化 |

## 🚀 快速开始

```html
<script src="liquid-glass.js"></script>
<liquid-glass src="背景图.png"></liquid-glass>
```

或直接打开 [`showcase.html`](./showcase.html)（推荐 Chrome / Edge，需 Chromium 内核支持 `backdrop-filter: url(#svg)`；Firefox 会降级为普通毛玻璃）。

在线体验：把仓库文件放到任意静态服务器即可，无构建步骤。

## 📁 文件结构

```
├── showcase.html          # 展示页：折射 + 果冻按压 + 体积光 + 光照互通 + 流体拖拽
├── demo.html              # 精简演示
├── liquid-glass.js        # 原版折射引擎（shuding/liquid-glass, MIT）
└── docs-算法与参数.md     # 完整算法文档：SDF fragment、归一化陷阱表、全部时间常数
```

## 🔧 核心原理（三行版）

1. **折射**：对玻璃内每个像素计算内缩矩形 SDF → `smoothStep` 得到位移场 → 归一化生成 `feImage` 贴图 → `feDisplacementMap` 弯曲真实背景
2. **果冻**：动态 `transform-origin`（指针反侧偏移 30%）+ 定向 scale = 非均匀局部拉伸；`exp(−5t)·cos(9t)` 弹簧回弹
3. **光照**：受光层 `radial-gradient` 圆心实时对齐光源方向，强度随距离线性衰减

详细算法、参数速查表与踩坑清单见 [docs-算法与参数.md](./docs-算法与参数.md)。

## ⚠️ 已知陷阱（血泪经验）

- SDF 写成 0.5/0.5 撑满元素 → 位移场恒为 1 → 归一化后 scale≈0 → **折射完全消失，变成塑料片**
- `maxScale` 归一化后忘了与 `feDisplacementMap` 的 scale 对应 → 折射强度差一倍
- 按钮按下"变暗"是因为内阴影压了顶部——下陷感应该只压底部，顶部留给增亮层

## 📄 授权

- 本仓库代码：MIT
- 折射引擎基于 [shuding/liquid-glass](https://github.com/shuding/liquid-glass)（MIT License, © Shu Ding）
- 动效标准参考 Apple WWDC25《Meet Liquid Glass》
