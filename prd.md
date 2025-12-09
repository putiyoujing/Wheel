

### 🏛️ 开发蓝图：核心架构

这个应用在逻辑上分为两层：**“生成器（Generator）”** 和 **“游戏本体（Game）”**。虽然它们最终都在一个 HTML 文件里，但思维上要分开处理。

#### 1\. 数据结构设计 (Data Schema)

在代码中，你需要一个核心对象来存储当前转盘的状态。无论是生成时还是导出后，都依赖这个数据结构：

```javascript
const wheelConfig = {
  title: "周末去哪玩", // 用户输入的标题
  themeId: "theme_cyberpunk", // 选中的主题ID
  totalChance: 100, // 校验值，必须为100
  sectors: [ // 扇形区域数据
    { text: "环球影城", percentage: 30, color: "#ff0000" }, // 颜色根据主题色板自动分配
    { text: "宅家睡觉", percentage: 50, color: "#ffffff" },
    { text: "去爬山", percentage: 20, color: "#0000ff" }
  ]
};
```

-----

### 🎨 关键功能实现细节

#### A. 视觉与主题系统 (CSS Variables)

  * **实现方式：** 在 `<style>` 标签的 `:root` 中定义变量。
  * **切换逻辑：** 当用户在配置界面选择“赛博朋克”时，JS 会修改 `body` 的 class，或者直接修改 CSS 变量的值。
  * **导出逻辑：** 导出时，将**当前选中的那组颜色值**硬编码写入导出文件的 `:root` 中，确保用户打开就是这个颜色。

#### B. 转盘绘制 (Canvas API)

  * **难点回顾：** 需要根据 `percentage` 计算弧度（Radian）。
  * **公式：** `endAngle = startAngle + (percentage / 100) * 2 * Math.PI`。
  * **文字处理：** 文字需要根据角度进行旋转（`context.rotate`）并居中绘制在扇形中间。对于太长的文字，需要做简单的截断或换行处理。

#### C. 音效集成 (Base64 Audio) 🎵

这是实现单文件的关键难点。

  * **资源准备：** 你需要准备两个短音频文件（`tick.mp3` - 旋转声，`win.mp3` - 胜利声）。
  * **编码：** 使用在线工具将 mp3 转为 **Base64 字符串**。
  * **代码实现：**
    ```javascript
    const soundClick = new Audio("data:audio/mp3;base64,SUQzBAAAAAAAI1RTU...");
    const soundWin = new Audio("data:audio/mp3;base64,//uQxAAAAAAAAAAAA...");

    // 播放时
    function playTick() {
        soundClick.currentTime = 0; // 重置进度
        soundClick.play();
    }
    ```
  * **注意：** 浏览器为了防止自动播放干扰，通常要求用户先与页面交互（点击一次）后才能播放声音。你的“开始转盘”按钮正好满足这个条件。

#### D. 撒花特效 (Canvas Particle System) 🎉

  * **方案：** 为了不依赖外部库（如 `canvas-confetti` CDN），你需要一段轻量级的原生 JS 粒子代码（大约 50-100 行）。
  * **逻辑：** 1.  创建一个全屏的透明 Canvas 覆盖在最上层。
    2\.  生成 100+ 个随机颜色的小方块/圆点对象。
    3\.  给它们赋予重力（y轴加速度）和初速度。
    4\.  使用 `requestAnimationFrame` 更新位置，直到它们掉出屏幕。

#### E. 导出引擎 (The Exporter) 💾

这是最神奇的一步。你需要编写一个函数 `exportGame()`。

  * **原理：** 拼接字符串。
  * **步骤：**
    1.  读取 `index.html` 的自身内容（或者你准备好的模板字符串）。
    2.  **注入数据：** 找到模板中的 `const gameData = null;`，把它替换为 `const gameData = ${JSON.stringify(wheelConfig)};`。
    3.  **注入状态：** 确保导出的 HTML 默认显示“游玩界面”而不是“配置界面”（可以通过设置 CSS `display: none` 实现）。
    4.  **生成文件：**
    <!-- end list -->
    ```javascript
    const blob = new Blob([finalHtmlContent], { type: "text/html" });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = "我的转盘.html";
    link.click();
    ```

-----

### 📝 开发步骤清单 (Checklist)

你可以按照这个顺序进行开发，一步步攻克：

1.  **静态页面搭建：**
      * 写好 HTML 结构（配置区、转盘区、模态框）。
      * 写好 CSS（布局、9种主题的配色变量）。
2.  **核心逻辑 - 输入端：**
      * 实现动态添加/删除选项。
      * 实现占比输入的实时校验（总和 100% 提示）。
3.  **核心逻辑 - 转盘端：**
      * 使用 Canvas 根据输入数据把饼图画出来。
      * 实现旋转动画（缓动效果）。
      * 实现停止位置计算（确保指针指在正确的扇区）。
4.  **增强体验：**
      * 加入 Base64 音效。
      * 加入 Canvas 撒花特效。
      * 制作自定义模态框（显示结果 + “再玩一次”按钮）。
5.  **导出功能开发：**
      * 编写“自身源码提取与注入”逻辑。
      * 测试导出的 HTML 文件是否能在断网环境下正常运行。

-----

### 💡 最后的提示

这个项目最大的难点在于\*\*“代码的自指（Quine）”\*\*——即程序需要读取并复制自身来生成新的程序。

**一个简单的技巧：**
你可以把所有的 CSS 和 JS 都写在 `index.html` 内部，不要用 `<link rel="stylesheet">` 或 `<script src="...">`。这样，`document.documentElement.outerHTML` 就能拿到几乎完整的当前页面代码，你只需要用正则替换掉里面的配置数据即可。
