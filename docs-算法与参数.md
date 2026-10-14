# 液态玻璃 Liquid Glass Web 实现文档

> 基于原版 [shuding/liquid-glass](https://github.com/shuding/liquid-glass)（MIT, 2025）逐字移植 + 苹果交互动效扩展。
> 完整算法、参数与时间常数，可直接搬到任意 Web 项目。可运行源码见 `showcase.html`。

---

## 一、液态玻璃折射效果（核心）

### 1.1 原理

玻璃本体是普通 DOM 元素，通过 **`backdrop-filter: url(#SVG滤镜)`** 让浏览器对实时渲染的背景做像素级位移——折射的是真实背景，永不糊、永不对不齐。

```xml
<feImage href="位移贴图.png" result="map"/>
<feDisplacementMap in="SourceGraphic" in2="map" xChannelSelector="R" yChannelSelector="G" scale="..."/>
```

### 1.2 位移贴图（fragment 函数）

```js
// 原版 fragment —— 所有玻璃共用，观感一致
function fragment(uv) {
  const ix = uv.x - 0.5;
  const iy = uv.y - 0.5;
  const distanceToEdge = roundedRectSDF(ix, iy, 0.3, 0.2, 0.6);  // 内缩 SDF
  const displacement = smoothStep(0.8, 0, distanceToEdge - 0.15);
  const scaled = smoothStep(0, 1, displacement);
  return { x: ix * scaled + 0.5, y: iy * scaled + 0.5 };
}
```

**关键点：**
- **SDF 必须内缩**（0.3/0.2）。写成 0.5/0.5 撑满 → 位移场恒为 1 → 归一化后 `scale≈0` → 折射消失（"固态塑料片"根因）。
- 效果：**中心平坦（放大感）→ 边缘一圈强烈弯曲**，苹果式"边缘折射圈"。
- 按压加厚：阈值改为 `-0.15 / strength`，按下 1.3 / 松开 1，重建贴图。

### 1.3 归一化与 scale

```js
maxScale = max(|raw|);
maxScale *= 0.5;            // 原版折半（勿删）
// 贴图像素 = raw / maxScale + 0.5，×255
// feDisplacementMap scale = maxScale / canvasDPI
```

| 陷阱 | 症状 |
|---|---|
| SDF 写 0.5/0.5 撑满 | 完全没有折射 |
| Firefox | 不支持 `backdrop-filter: url(#id)` |

### 1.4 滤镜链与玻璃本体（原版同款）

```css
backdrop-filter: url(#id_filter) blur(0.25px) contrast(1.2) brightness(1.05) saturate(1.1);

.glass {
  background: rgba(255,255,255,0.08);    /* 白 tint，偏白不偏灰 */
  box-shadow: 0 4px 8px rgba(0,0,0,0.25),
              0 -10px 25px inset rgba(0,0,0,0.15); /* 底部内阴影（"厚"感来源）*/
}
```

---

## 二、体积光（从整个按钮轮廓发出）

**不是点光源**：wrapper 以按钮中心为原点、按按钮 2.6 倍包络做**椭圆缩放**（药丸形 → 扁椭圆），光核 + 光束都从整个轮廓涌出。三层：光核（径向渐变）+ 旋转光束（conic-gradient 扇形条 + 径向 mask）+ `screen` 混合。

```js
function emitVolumetricLight(btn) {
  const r = btn.getBoundingClientRect();
  vl.style.left = (r.left + r.width/2) + 'px';
  vl.style.top  = (r.top + r.height/2) + 'px';
  // 关键：椭球比例贴合按钮轮廓
  vl.style.transform =
    `scale(${r.width * 2.6 / 900}, ${r.height * 2.6 / 900})`;
  vl.classList.remove('play'); void vl.offsetWidth; vl.classList.add('play');
}
```

```css
.vl-core {  /* 光核 900×900 径向渐变，mix-blend-mode: screen */ }
.vl-rays {  /* 1100×1100 conic-gradient 多条不均匀光束 + radial mask 衰减 */ }
.vl.play { animation: vlLife 1.4s cubic-bezier(0.16,1,0.3,1) forwards; }
/* 光核 0.35→1.15 扩散；光束 6s/圈自转、1.4s 内旋转 80° 同时淡出 */
```

**时间常数：寿命 1.4s（expo-out），光束自转 6s/圈，淡出期旋转 80°。**

---

## 三、光晕互相影响（照亮系统）

受光层**朝向光源**，距离线性衰减：

```css
.lit-glow {
  background: radial-gradient(circle at var(--lx,50%) var(--ly,50%),
    rgba(255,255,255,0.9) 0%, rgba(255,255,255,0.25) 40%, transparent 70%);
  mix-blend-mode: soft-light; opacity: 0; transition: opacity 0.25s;
}
```

```js
function illuminate(srcEl, cx, cy, radius, base) {
  allGlass.filter(g => g !== srcEl).forEach(g => {
    const r = g.getBoundingClientRect();
    const d = Math.hypot(r.left + r.width/2 - cx, r.top + r.height/2 - cy);
    setLit(g, cx, cy, base * Math.max(0, 1 - d / radius));
  });
}
```

| 光源 | 事件 | 范围 |
|---|---|---|
| 胶囊按钮 | 按下/按住拖动 | 700~900px，强度 0.9~1.0 |
| 拖动玻璃 | 移动/滑行 | 460px，0.25~0.85 随速度 |

---

## 四、流体物理动画

### 4.1 按下 = 果冻被压扁下陷（不是变小）

两个分量叠加：

```js
// ① 压扁下陷：scaleX 变宽、scaleY 变扁 + 内阴影加深（按下期间持续存在）
el.style.boxShadow = '0 2px 4px rgba(0,0,0,0.3), inset 0 8px 18px rgba(0,0,0,0.22)';
const sx = 1.05, sy = 0.93;   // 宽 5%、扁 7% —— 视觉是"被按瘪"不是"缩小"

// ② 按住拖动：朝指针方向的【局部】拉伸
// 技巧：transform-origin 移到指针反侧 30% 处再放大 → 指针那半边鼓得多、另一侧不动
el.style.transformOrigin = (50 - ux*30) + '% ' + (50 - uy*30) + '%';
const k = Math.min(0.14, dist / 900);
el.style.transform =
  `translate(${ux*k*26}px, ${uy*k*26}px) scale(${sx*(1+k)}, ${sy*(1+k*0.75)})`;
```

松手：`jellyBack(el, k*1.2, 500)` 回弹 + 复位 origin + 恢复外阴影。

### 4.2 果冻摇晃回弹（时间）

```js
function springEase(t) { return 1 - Math.exp(-5 * t) * Math.cos(9 * t); }
function jellyBack(el, from, dur = 500) { }   // 500ms，一个过冲周期
```

### 4.3 拖拽 / 滑行 / 运动发光

```js
vx = vx * 0.7 + (e.clientX - lastX) / dt * 16 * 0.3;   // 速度平滑

// 拖拽拉伸：沿速度方向、体积守恒、任意角度
const k = Math.min(0.16, speed / 420);
const deg = Math.atan2(vy, vx) * 180 / Math.PI;
transform = `rotate(${deg}deg) scale(${1+k}, ${1-k*0.45}) rotate(${-deg}deg)`;

// 惯性滑行：0.94/帧，停后补 500ms 余晃
// 运动发光：glow = min(0.85, speed/26)
box-shadow 加: 0 0 ${18 + speed*2.2}px rgba(255,255,255,${0.12 + glow*0.5})
```

---

## 五、参数速查表

| 参数 | 值 | 说明 |
|---|---|---|
| fragment 形状 | SDF(0.3, 0.2, r=0.6) | 所有玻璃一致 |
| 折射阈值 | `-0.15`（按压 ÷1.3） | 按压加厚 |
| maxScale 折半 | `×0.5` | 原版手感 |
| 滤镜链 | blur 0.25 / contrast 1.2 / brightness 1.05 / saturate 1.1 | 原版同款 |
| 体积光 | 按钮 2.6 倍包络椭圆 | 整个轮廓发光，非点光源 |
| 体积光寿命 | 1400ms，光束 6s/圈 | expo-out |
| 按压压扁 | scale(1.05, 0.93) + 内阴影加深 | 果冻下陷感 |
| 局部拉伸 | origin 反侧偏移 30%，k=min(0.14, dist/900) | 朝指针鼓出 |
| 摇晃回弹 | 500ms, exp(−5t)·cos(9t) | 一个过冲周期 |
| 拖拽拉伸上限 | 16% | 体积守恒 |
| 滑行摩擦 | 0.94/帧 | |
| 运动发光 | `min(0.85, speed/26)` | 半径 18+2.2·speed px |

## 六、兼容性

- ✅ Chrome / Edge（`backdrop-filter: url(#svg)` 仅 Chromium）
- ❌ Firefox：降级 blur 毛玻璃
- 移动端：`touch-action: none` + Pointer Events

## 七、文件清单

| 文件 | 说明 |
|---|---|
| `liquid-glass.js` | 原版脚本（MIT, shuding）|
| `demo.html` | 三件套完整演示 |
| `showcase.html` | 展示页（本文档效果的可运行源码）|
