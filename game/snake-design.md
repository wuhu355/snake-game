# 贪吃蛇游戏设计文档

## 1. 概述

基于 HTML5 Canvas 的单文件网页贪吃蛇游戏，无需外部依赖，浏览器直接打开即可运行。

---

## 2. 项目结构

```
game/
  snake.html          # 游戏完整代码（HTML + CSS + JS）
  snake-design.md     # 本设计文档
```

单文件内部分层：

```
snake.html
  <style>             # CSS 样式
  <body>              # HTML 结构
  <script>            # JS 游戏逻辑（IIFE 包裹）
    DOM 引用            # canvas、ctx、scoreDisplay、messageDisplay
    常量定义            # 画布尺寸、格子大小、速度
    状态变量            # snake、direction、food、score、gameState
    音效模块            # Web Audio API 吃食物 + 游戏结束音效
    初始化模块          # initGame()、placeFood()
    游戏逻辑模块        # update()、endGame()
    渲染模块            # draw()、roundRect()
    游戏主循环          # gameLoop()、startGameLoop()
    输入处理模块        # 键盘 + 触摸
    事件绑定 & 启动
```

---

## 3. 架构设计

### 3.1 整体架构

```
键盘/触摸事件 --> Input 层 --> Update 层 --> Draw 层 --> Canvas
                  |             |              |
              nextDirection   snake/food    clearRect
                              score         fill/arc
                              gameState     shadow
```

- **Input 层**: 监听键盘和触摸事件，转换为方向指令，存入 nextDirection
- **Update 层**: 每 150ms 执行移动、碰撞检测、吃食物判定
- **Draw 层**: 每帧绘制画布，食物带脉动动画

### 3.2 requestAnimationFrame vs setInterval

| 维度 | setInterval | requestAnimationFrame |
|------|------------|----------------------|
| 帧同步 | 与屏幕刷新无关 | 与 60fps 刷新率同步 |
| 后台标签页 | 继续执行，浪费资源 | 自动暂停 |
| 移动时机控制 | 天然匹配固定间隔 | 需手动累计时间差 |

采用 rAF + 手动计时混合方案：rAF 驱动渲染保证流畅，循环内累计时间差，达到 150ms 阈值时执行逻辑更新。

---

## 4. 数据模型

### 4.1 常量

| 常量 | 值 | 说明 |
|------|-----|------|
| CANVAS_WIDTH | 600 | 画布宽度 (px) |
| CANVAS_HEIGHT | 400 | 画布高度 (px) |
| CELL_SIZE | 20 | 每格边长 (px) |
| COLS | 30 | 网格列数 = 600/20 |
| ROWS | 20 | 网格行数 = 400/20 |
| BASE_SPEED | 150 | 移动间隔 (ms) |

### 4.2 状态变量

| 变量 | 类型 | 初始值 | 说明 |
|------|------|--------|------|
| snake | Array<{x,y}> | 3 节水平排列 | 蛇身坐标，索引 0 为蛇头 |
| direction | {x,y} | {1,0} | 当前帧移动方向 |
| nextDirection | {x,y} | {1,0} | 下帧要应用的方向（防反向队列） |
| food | {x,y} | 随机空白格 | 食物坐标 |
| score | number | 0 | 当前分数 |
| gameState | string | 'idle' | idle / playing / over |
| gameLoopId | number|null | null | rAF 返回的帧 ID |
| lastMoveTime | number | 0 | 上次移动的时间戳 |

### 4.3 坐标系统

```
(0,0) --------------------> (29,0)
 |                             |
 |   每个格子 20px * 20px       |
 |                             |
 v                             v
(0,19) -------------------> (29,19)
```

- x 轴向右: 0 ~ 29
- y 轴向下: 0 ~ 19
- 格子坐标 (gx, gy) 对应画布像素 (gx*20, gy*20)

---

## 5. 游戏状态机

三个状态:

```
idle --> playing --> over
  ^                    |
  |____________________|
     (按空格)
```

| 状态 | 画面 | 键盘响应 | 消息提示 |
|------|------|---------|---------|
| idle | 初始蛇 + 食物 | 仅空格 | "按空格键开始游戏" |
| playing | 蛇移动，食物刷新 | 方向键 + WASD | 无（得分更新） |
| over | 碰撞瞬间冻结 | 仅空格 | "游戏结束，得分：xxx" |

状态转换:

| 转换 | 触发 | 动作 |
|------|------|------|
| idle -> playing | 空格 | initGame() -> gameState='playing' -> startGameLoop() |
| playing -> over | 碰墙或碰自身 | gameState='over' -> cancelAnimationFrame() -> 播放音效 |
| over -> playing | 空格 | 同 idle -> playing |

---

## 6. 核心算法

### 6.1 update() 流程

```
1. direction = nextDirection         -- 应用排队方向
2. newHead = head + direction        -- 计算新蛇头
3. 判断 newHead 是否出界 --> 是: endGame()
4. 构建 bodySet, 排除尾巴坐标
   判断 newHead 是否在 bodySet 中 --> 是: endGame()
5. snake.unshift(newHead)           -- 插入新蛇头
6. newHead == food ?
    是: score += 10, playEatSound(), placeFood(), 不删尾巴 (+1 节)
    否: snake.pop()                  -- 删除尾巴
```

### 6.2 自身碰撞优化

使用 Set 代替数组遍历查重:

```js
const bodySet = new Set(snake.map(s => `${s.x},${s.y}`));
bodySet.delete(`${tail.x},${tail.y}`);  // 排除即将移走的尾巴
```

- 时间复杂度: O(n) + O(1) vs 朴素 O(n^2)
- 排除尾巴: 防止蛇头移动到尾巴当前位置时误判（尾巴即将移走）

### 6.3 防反向设计

**第一层 - 按键时同步拦截:**

```js
// 新方向和当前实际方向完全相反则丢弃
if (newDir.x === -direction.x && newDir.y === -direction.y) return;
nextDirection = newDir;
```

**第二层 - 方向队列:**

每帧只存一个 nextDirection（最近一次有效按键），update() 开头应用。反转判断基准是当前已应用的 direction，而非 nextDirection。

快速连按场景:

```
蛇向右走: direction = {1,0}
按键 1: 上 --> nextDirection = {0,-1}   [通过]
按键 2: 左 --> -1===-1 && 0===-0 --> true --> [拦截]
结果: 蛇向上走，不会反向
```

### 6.4 placeFood() 算法

```
1. 构建已占用集合: Set(snake 坐标)
2. 遍历 30*20 = 600 个格子，收集未占用者
3. 从空闲格子中随机选一个
4. 若空闲格子数 = 0（蛇占满画布），不更新 food
```

---

## 7. 渲染设计

### 7.1 绘制层次（从底到顶）

```
1. clearRect         清空画布
2. 网格线             0.03 透明度白色线条
3. 食物
     外光晕           红色半透明脉冲光晕
     主体             红色圆形
     高光             白色小圆点
4. 蛇身
     身体段           圆角矩形，绿渐变
     蛇头             亮绿 + 发光阴影 + 眼睛
```

### 7.2 蛇身渐变色

```js
const ratio = index / snake.length;
r = 30 + ratio * 20       // 30 -> 50
g = 180 - ratio * 60      // 180 -> 120
b = 130 - ratio * 50      // 130 -> 80
```

蛇头: rgb(30,180,130) 明亮青绿
蛇尾: rgb(50,120,80) 较暗深绿

### 7.3 食物脉动动画

```js
const pulse = 1 + 0.12 * Math.sin(Date.now() / 200);
```

- Date.now() 驱动，游戏暂停时动画不停
- 幅度 12%，周期约 1.26 秒

### 7.4 蛇头眼睛朝向

| 方向 | 左眼偏移 | 右眼偏移 |
|------|---------|---------|
| 右 (1,0) | (cx+3, cy-4) | (cx+3, cy+4) |
| 左 (-1,0) | (cx-3, cy-4) | (cx-3, cy+4) |
| 上 (0,-1) | (cx-4, cy-3) | (cx+4, cy-3) |
| 下 (0,1) | (cx-4, cy+3) | (cx+4, cy+3) |

---

## 8. 输入系统

### 8.1 键盘映射

| 按键 | 方向 | 要求 |
|------|------|------|
| ArrowUp / W | (0,-1) 上 | playing |
| ArrowDown / S | (0,1) 下 | playing |
| ArrowLeft / A | (-1,0) 左 | playing |
| ArrowRight / D | (1,0) 右 | playing |
| Space | 开始/重开 | idle / over |

方向键和空格均 preventDefault 阻止页面滚动。

### 8.2 触摸滑动

```
touchstart: 记录起始坐标 (sx, sy)
touchend:   计算 dx, dy
            |dx|>|dy| --> 水平滑动
            |dy|>|dx| --> 垂直滑动
            max(|dx|,|dy|)<20px --> 忽略
```

---

## 9. 音效系统

Web Audio API 程序化生成，无需外部文件。

| 音效 | 波形 | 频率 | 持续 | 音量 |
|------|------|------|------|------|
| 吃食物 | 方波 square | 440->880Hz 上升 | 120ms | 0.15 |
| 游戏结束 | 锯齿波 sawtooth | 300->100Hz 下降 | 500ms | 0.18 |

AudioContext 懒初始化（首次用户交互时创建），符合浏览器自动播放策略。try-catch 包裹，失败静默降级。

---

## 10. 视觉风格

| 元素 | 配色/样式 |
|------|----------|
| 页面背景 | 深蓝渐变 #1a1a2e -> #16213e -> #0f3460 |
| 计分板 | 半透明玻璃 rgba(255,255,255,0.08) + backdrop-filter blur |
| 画布底色 | #0d0d0d 近黑色 |
| 网格线 | rgba(255,255,255,0.03) 极淡 |
| 蛇头 | #4ecca3 + 绿色发光阴影 |
| 蛇身 | 亮绿渐变深绿 |
| 食物 | #e74c3c 红色 + 红色外光晕 |
| 分数数字 | #4ecca3 |
| 结束提示 | #e74c3c |

---

## 11. 边界情况

| 场景 | 处理方式 |
|------|---------|
| 蛇占满画布 | placeFood() 检测无空闲格子时直接返回 |
| 快速连按反向 | 两层防护：同步拦截 + 每帧单方向队列 |
| 游戏结束按键 | gameState !== 'playing' 时忽略方向键 |
| 页面失去焦点 | rAF 自动暂停 |
| 触摸误触 | 滑动距离阈值 20px |
| AudioContext 失败 | try-catch 静默降级 |

---

## 12. 关键决策

**为什么用 IIFE?**
避免全局命名空间污染，所有变量和函数封闭在闭包内。

**为什么 initGame() 不清空 messageDisplay?**
initGame() 在 idle（需显示"按空格开始"）和 playing（需清空消息）两种场景都会被调用。消息管理职责归属状态转换点 startGame() 和 endGame()。

**为什么 lastMoveTime 初始值为 0?**
设为 0 确保首帧 timestamp-0 >= 150 恒成立，蛇立即开始移动，无需等待首个间隔周期。

---

## 13. 扩展方向

- 难度递增（随分数提高缩短移动间隔）
- localStorage 持久化最高分排行榜
- 障碍物模式
- 双人模式（WASD vs 方向键）
- 移动端虚拟方向键
- 主题切换（经典绿、霓虹、像素风）
