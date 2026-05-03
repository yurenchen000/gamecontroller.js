# POV Axes 支持

## 一、背景介绍

POV（Point of View）轴常见于飞行摇杆、赛车方向盘等外设，物理上就是十字键（D-Pad）。与传统摇杆不同，POV 轴输出的是**离散方向值**而非连续模拟量。

典型 POV 轴数据（以 axes[4] 为例）：

```
[-1, -0.7142857313156128, -0.4285714030265808, -0.1428571343421936,
 0.14285719394683838, 0.4285714626312256, 0.7142857313156128, 1]
```

分别对应 8 个方向：

| 值 | 方向 |
|----|------|
| -1 | up |
| -0.714... | up-right |
| -0.428... | right |
| -0.142... | down-right |
| 0.142... | down |
| 0.428... | down-left |
| 0.714... | left |
| 1 | up-left |

> 取值 在 [-1, 1] 区间内，表示操作方向，值相对固定，来自硬件固件映射。
> 取值 在 [-1, 1] 范围外，表示空闲。

---

## 二、设计思路

### 2.1 接口设计

参考已有摇杆事件 `up0`/`down0` 的风格，POV 轴使用 `povN.direction` 格式：

```javascript
gamepad.before('pov4.up',   callback)
       .after ('pov4.up',   callback);

gamepad.before('pov4.down', callback)
       .after ('pov4.down', callback);

gamepad.before('pov4.left',  callback)
       .after ('pov4.left',  callback);

gamepad.before('pov4.right', callback)
       .after ('pov4.right', callback);
```

- `pov4` 表示使用 `axes[4]` 作为 POV 轴解析
- `before`：方向刚按下时触发（类似 `onkeydown`）
- `after`：方向松开时触发（类似 `onkeyup`）
- 支持任意 axes 索引，如 `pov0`、`pov2`、`pov6` 等

### 2.2 斜方向处理模式

POV 轴有 8 个方向，其中 4 个为斜方向（up-right、down-right、down-left、up-left）。斜方向是否触发 `up/down/left/right` 事件，提供两种模式：

#### 模式 A：`oblique_as_both`（斜方向视为正方向叠加）

斜方向会同时触发两个正方向事件。

示例流程：
```
空闲 → 右上：触发 up.before + right.before
右上 → 上：  触发 right.after + up.before（right 释放，up 仍在）
上 → 右上：  触发 up.after + right.before（up 释放，right 按下）
```

#### 模式 B：`oblique_as_none`（斜方向不触发正方向）

斜方向完全被忽略，只处理 4 个正方向。

示例流程：
```
空闲 → 右上：无事件
右上 → 上：  触发 up.before（不触发 right 相关事件）
上 → 右上：  触发 up.after（不触发 right 相关事件）
```

默认模式为 `oblique_as_none`，可通过 `set()` 方法切换：

```javascript
gamepad.set('obliqueMode', 'oblique_as_both');  // 或 'oblique_as_none'
```

---

## 三、实现过程

### 3.1 新增状态字段

在 `src/gamepad.js` 的 `gamepadPrototype` 中新增：

```javascript
povAxes: [],      // 注册为 POV 的 axes 索引列表，如 [4, 6]
povActions: {},    // POV 事件回调存储，结构：{ povIndex: { up/down/left/right: { before/action/after } } }
povState: {},      // 当前 POV 方向状态，结构：{ povIndex: ['up', 'right'] }（数组，支持斜方向展开）
obliqueMode: 'oblique_as_none',  // 斜方向处理模式
```

### 3.2 POV 值 → 方向映射

`povValueToDirection(value)`：将 axes 原始值映射到 8 方向字符串。

核心逻辑：预定义 8 个参考值和方向数组，计算输入值与所有参考值的绝对差，取最小差对应的方向。

```javascript
povValueToDirection: function(value) {
  if (value >= -1 && value <= 1) {
    const povValues = [-1, -0.7142857313156128, -0.4285714030265808,
      -0.1428571343421936, 0.14285719394683838, 0.4285714626312256,
      0.7142857313156128, 1];
    const directions = ['up', 'up-right', 'right', 'down-right',
      'down', 'down-left', 'left', 'up-left'];
    // 找最小差 → 返回对应方向
  }
  return 'center';
}
```

### 3.3 斜方向展开

`expandObliqueDirection(direction)`：根据 `oblique_as_both` 模式，将斜方向展开为正方向数组。

```javascript
expandObliqueDirection: function(direction) {
  const mapping = {
    'up': ['up'],
    'up-right': ['up', 'right'],
    'right': ['right'],
    'down-right': ['down', 'right'],
    'down': ['down'],
    'down-left': ['down', 'left'],
    'left': ['left'],
    'up-left': ['up', 'left']
  };
  return mapping[direction] || [direction];
}
```

### 3.4 POV 状态检测

`checkPovAxes(gp, modifier)`：在每帧 `checkStatus()` 中调用，检测所有已注册 POV 轴的状态变化。

流程：
1. 遍历 `povAxes` 中每个 POV 索引
2. 读取对应 axes 值，调用 `povValueToDirection()` 得到原始方向
3. 根据 `obliqueMode` 决定 `newDirections`（空数组 / 展开后的正方向数组）
4. 对比 `oldDirections`（上次状态）与 `newDirections`：
   - 在 old 不在 new → 触发 `after`
   - 在 new 不在 old → 触发 `before`
   - new 中所有方向 → 触发 `action`
5. 更新 `povState[povIndex]`

### 3.5 事件绑定

在 `associateEvent()` 中新增 `povN.direction` 格式匹配：

```javascript
} else if (eventName.match(/^pov(\d+)\.(up|down|left|right)$/)) {
  const matches = eventName.match(/^pov(\d+)\.(up|down|left|right)$/);
  const povIndex = parseInt(matches[1]);
  const direction = matches[2];

  if (!this.povActions[povIndex]) {
    this.initPovActions(povIndex);  // 初始化事件空函数
  }
  if (this.povAxes.indexOf(povIndex) === -1) {
    this.povAxes.push(povIndex);     // 自动注册为 POV 轴
  }
  this.povActions[povIndex][direction][type] = callback;
}
```

绑定事件时自动将对应 axis 注册为 POV 轴，无需手动注册。

### 3.6 配置支持

`set()` 方法新增 `obliqueMode` 属性支持：

```javascript
set: function(property, value) {
  const properties = ['axeThreshold', 'obliqueMode'];
  if (properties.indexOf(property) >= 0) {
    if (property === 'obliqueMode' && value !== 'oblique_as_both' && value !== 'oblique_as_none') {
      error(MESSAGES.INVALID_VALUE);
      return;
    }
    this[property] = value;
  } else {
    error(MESSAGES.INVALID_PROPERTY);
  }
}
```

同步更新 `src/constants.js`，新增 `INVALID_VALUE` 提示。

---

## 四、使用示例

```javascript
gameControl.on('connect', function(gamepad) {

  // 可选：设置斜方向模式（默认 oblique_as_none）
  gamepad.set('obliqueMode', 'oblique_as_both');

  // 绑定 POV 轴（axes[4]）方向事件
  gamepad.before('pov4.up',    () => console.log('POV up pressed'))
         .after ('pov4.up',    () => console.log('POV up released'));

  gamepad.before('pov4.down',  () => console.log('POV down pressed'))
         .after ('pov4.down',  () => console.log('POV down released'));

  gamepad.before('pov4.left',  () => console.log('POV left pressed'))
         .after ('pov4.left',  () => console.log('POV left released'));

  gamepad.before('pov4.right', () => console.log('POV right pressed'))
         .after ('pov4.right', () => console.log('POV right released'));
});
```

---

## 五、修改文件清单

| 文件 | 修改内容 |
|------|----------|
| `src/gamepad.js` | 新增 POV 状态字段、povValueToDirection、checkPovAxes、expandObliqueDirection、initPovActions，修改 associateEvent 和 set 方法 |
| `src/constants.js` | 新增 `INVALID_VALUE` 常量 |
| `examples/pov_test.htm` | POV 测试示例页面 |
| `support_pov_axes.md` | 本文档 |
