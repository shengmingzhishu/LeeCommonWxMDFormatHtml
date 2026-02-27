# 小红书风格 MD 排版工具 - 详细实现方案

## 项目概述

在现有 `md-custom-style-youhua-page-master-img.html` 基础上，新增小红书风格主题、智能拆分和批量保存功能。

## 一、需求总结

### 1.1 核心功能
- ✅ 小红书风格主题系统
- ✅ 智能内容拆分（基于像素高度计算）
- ✅ 批量图片生成和保存
- ✅ 右侧双列网格预览
- ✅ 作者和关注语自定义
- ✅ 保持所有现有功能

### 1.2 布局结构

```
┌─────────────────────────────────────────────────────────────┐
│  页面主体                                                     │
│                                                              │
│  ┌──────────────────────┬──────────────────────────────────┐│
│  │   左侧：输入设置区   │   右侧：图片预览区（双列网格）    ││
│  │                      │                                  ││
│  │  ┌────────────────┐  │  ┌──────────┬──────────┐        ││
│  │  │ 小红书设置区域 │  │  │  主图    │  内容图1 │        ││
│  │  │ (主题/作者/关注)│  │  │ 1080x1440│1080x1440 │        ││
│  │  └────────────────┘  │  └──────────┴──────────┘        ││
│  │                      │  ┌──────────┬──────────┐        ││
│  │  ┌────────────────┐  │  │  内容图2 │  内容图3 │        ││
│  │  │ 标题样式区域   │  │  │ 1080x1440│1080x1440 │        ││
│  │  │ (粘性置顶)    │  │  └──────────┴──────────┘        ││
│  │  └────────────────┘  │  ┌──────────┬──────────┐        ││
│  │                      │  │  内容图4 │  尾图    │        ││
│  │  ┌────────────────┐  │  │ 1080x1440│1080x1440 │        ││
│  │  │ 引用样式区域   │  │  └──────────┴──────────┘        ││
│  │  └────────────────┘  │  ...滚动查看更多...              ││
│  │                      │                                  ││
│  │  ┌────────────────┐  │                  ┌──────────┐  ││
│  │  │ 段落样式区域   │  │                  │批量保存  │  ││
│  │  └────────────────┘  │                  │按钮(悬停)│  ││
│  │                      │                  └──────────┘  ││
│  │  ┌────────────────┐  │                                  ││
│  │  │ MD内容输入框  │  │                                  ││
│  │  │ (自动高度+滚动)│  │                                  ││
│  │  └────────────────┘  │                                  ││
│  └──────────────────────┴──────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 1.3 图片规格
- **尺寸**：1080x1440（3:4比例）
- **格式**：PNG
- **质量**：高清（scale: 2）
- **命名**：01-主图.png, 02-内容1.png, 03-内容2.png...

### 1.4 文章结构解析
```
第1行 → 标题（主图使用）
第2行 → 副标题（内容图使用）
第3行 → 摘要（主图或内容图使用）
第4行开始 → 正文内容（智能拆分为多张内容图）
```

## 二、功能模块设计

### 2.1 小红书设置区域（新增）

#### 2.1.1 位置
在样式设置区域最上方，粘性置顶

#### 2.1.2 HTML结构
```html
<div class="style-section" id="xiaohongshu-section">
    <h3 class="style-section-title">🌸 小红书设置</h3>
    
    <!-- 主题切换 -->
    <div class="style-controls">
        <label>主题风格：
            <button id="theme-prev-btn">◀</button>
            <span id="current-theme-name">清新日系</span>
            <button id="theme-next-btn">▶</button>
        </label>
        
        <!-- 主题下拉选择 -->
        <select id="theme-select" onchange="applyTheme(this.value)">
            <option value="theme-1">清新日系</option>
            <option value="theme-2">文艺复古</option>
            <option value="theme-3">简约现代</option>
            <option value="theme-4">可爱甜美</option>
            <option value="theme-5">暗黑酷感</option>
        </select>
    </div>
    
    <!-- 作者设置 -->
    <div class="style-controls">
        <label>作者名称：
            <input type="text" id="author-input" value="北屿的蓝色" 
                   placeholder="输入作者名称" style="width: 150px;">
        </label>
        <label>
            <input type="checkbox" id="show-author" checked>显示作者
        </label>
    </div>
    
    <!-- 关注语设置 -->
    <div class="style-controls">
        <label>关注语：
            <input type="text" id="follow-text-input" value="欢迎关注，点个好运赞吧~" 
                   placeholder="输入关注语" style="width: 200px;">
        </label>
    </div>
    
    <!-- 显示控制 -->
    <div class="style-controls">
        <label>
            <input type="checkbox" id="show-follow-main" checked>主图显示
        </label>
        <label>
            <input type="checkbox" id="show-follow-content-header" checked>内容图页眉
        </label>
        <label>
            <input type="checkbox" id="show-follow-content-footer" checked>内容图页脚
        </label>
        <label>
            <input type="checkbox" id="show-follow-end" checked>尾图显示
        </label>
    </div>
</div>
```

#### 2.1.3 主题配置
```javascript
const themes = {
    'theme-1': {
        name: '清新日系',
        h1Style: 'style-4',        // 清新治愈风
        h2Style: 'style-4',
        h3h6Style: 'heading-style-1',
        font: 'template-style-7',  // 苹方
        background: '#eaf6fa',     // 薄荷绿
        quoteStyle: 'quote-style-4', // 温馨提示
        quoteSymbol: 'symbol-1',   // 💡提示
        textColor: '#333333',
        accentColor: '#6BCF7F'
    },
    'theme-2': {
        name: '文艺复古',
        h1Style: 'style-2',        // 复古衬线风
        h2Style: 'style-2',
        h3h6Style: 'heading-style-2',
        font: 'template-style-2',  // 仿宋
        background: '#e6e2d3',     // 暖米色
        quoteStyle: 'quote-style-3', // 经典引用
        quoteSymbol: 'symbol-5',   // 📖引用
        textColor: '#4A4A4A',
        accentColor: '#D4A574'
    },
    'theme-3': {
        name: '简约现代',
        h1Style: 'style-1',        // 简约高级风
        h2Style: 'style-1',
        h3h6Style: 'heading-style-1',
        font: 'template-style-1',  // 微软雅黑
        background: '#f8f9fa',     // 象牙白
        quoteStyle: 'quote-style-1', // 基础引用
        quoteSymbol: 'symbol-0',   // 无符号
        textColor: '#333333',
        accentColor: '#333333'
    },
    'theme-4': {
        name: '可爱甜美',
        h1Style: 'style-7',        // 轻奢质感风
        h2Style: 'style-7',
        h3h6Style: 'heading-style-9',
        font: 'template-style-5',  // 幼圆
        background: '#fdf2f8',     // 柔粉色
        quoteStyle: 'quote-style-13', // 引号引用
        quoteSymbol: 'symbol-7',   // ✨亮点
        textColor: '#FF69B4',
        accentColor: '#FFB6C1'
    },
    'theme-5': {
        name: '暗黑酷感',
        h1Style: 'style-9',        // 暗黑酷感风
        h2Style: 'style-9',
        h3h6Style: 'heading-style-5',
        font: 'template-style-10', // Arial
        background: '#d4e6f1',     // 湖水蓝
        quoteStyle: 'quote-style-15', // 渐变引用
        quoteSymbol: 'symbol-10',  // 🔥热门
        textColor: '#1A1A1A',
        accentColor: '#FF4500'
    }
};
```

### 2.2 智能拆分算法（核心）

#### 2.2.1 常量定义
```javascript
const IMAGE_CONFIG = {
    width: 1080,
    height: 1440,
    padding: 60,
    
    // 页眉页脚高度
    headerHeight: 80,
    footerHeight: 60,
    
    // 标题区域高度
    titleHeight: 120,      // H1标题
    subtitleHeight: 60,    // 副标题
    summaryHeight: 80,     // 摘要
    
    // 可用内容高度
    get availableHeight() {
        return this.height - this.padding * 2 - this.headerHeight - this.footerHeight;
    }
};

// 字体高度映射表（字号 → 行高）
const FONT_HEIGHT_MAP = {
    '14px': 22,
    '16px': 26,
    '18px': 30,
    '20px': 34,
    '24px': 40,
    '28px': 46,
    '32px': 52,
    '36px': 58,
    '40px': 64,
    '48px': 76,
    '56px': 88,
    '64px': 100
};

// 标题高度映射表
const HEADING_HEIGHT_MAP = {
    'h1': 120,
    'h2': 80,
    'h3': 60,
    'h4': 50,
    'h5': 45,
    'h6': 40
};
```

#### 2.2.2 元素高度计算函数
```javascript
/**
 * 计算文本元素的高度
 * @param {string} text - 文本内容
 * @param {string} fontSize - 字号
 * @param {number} maxWidth - 最大宽度
 * @param {string} fontFamily - 字体
 * @returns {number} 高度（像素）
 */
function calculateTextHeight(text, fontSize, maxWidth, fontFamily) {
    const lineHeight = FONT_HEIGHT_MAP[fontSize] || 26;
    const charWidth = estimateCharWidth(fontSize, fontFamily);
    const charsPerLine = Math.floor(maxWidth / charWidth);
    const lines = Math.ceil(text.length / charsPerLine);
    return lines * lineHeight;
}

/**
 * 估算字符宽度
 * @param {string} fontSize - 字号
 * @param {string} fontFamily - 字体
 * @returns {number} 单个字符宽度（像素）
 */
function estimateCharWidth(fontSize, fontFamily) {
    const baseSize = parseInt(fontSize);
    const widthRatio = {
        'Microsoft YaHei': 0.6,
        'SimSun': 0.55,
        'KaiTi': 0.58,
        'LiSu': 0.65,
        'YouYuan': 0.62,
        'FangSong': 0.55,
        'PingFang SC': 0.6,
        'Arial': 0.5,
        'Georgia': 0.55,
        'Times New Roman': 0.5
    };
    return baseSize * (widthRatio[fontFamily] || 0.6);
}

/**
 * 计算图片元素的高度
 * @param {number} originalWidth - 原始宽度
 * @param {number} originalHeight - 原始高度
 * @param {number} maxWidth - 最大宽度
 * @returns {number} 缩放后的高度（像素）
 */
function calculateImageHeight(originalWidth, originalHeight, maxWidth) {
    const ratio = originalHeight / originalWidth;
    return maxWidth * ratio;
}

/**
 * 计算引用块的高度
 * @param {string} text - 引用文本
 * @param {string} fontSize - 字号
 * @param {number} maxWidth - 最大宽度
 * @param {boolean} hasSymbol - 是否有符号
 * @returns {number} 高度（像素）
 */
function calculateQuoteHeight(text, fontSize, maxWidth, hasSymbol) {
    const lineHeight = FONT_HEIGHT_MAP[fontSize] || 26;
    const textHeight = calculateTextHeight(text, fontSize, maxWidth - 40, '');
    const padding = 30;
    const symbolHeight = hasSymbol ? 30 : 0;
    return textHeight + padding * 2 + symbolHeight;
}

/**
 * 计算段落块的高度
 * @param {string} text - 段落文本
 * @param {string} fontSize - 字号
 * @param {number} maxWidth - 最大宽度
 * @param {boolean} hasIndent - 是否首行缩进
 * @param {string} background - 背景颜色
 * @returns {number} 高度（像素）
 */
function calculateParagraphHeight(text, fontSize, maxWidth, hasIndent, background) {
    const lineHeight = FONT_HEIGHT_MAP[fontSize] || 26;
    const textHeight = calculateTextHeight(text, fontSize, maxWidth, '');
    const padding = background !== 'none' ? 30 : 0;
    const margin = 20;
    return textHeight + padding * 2 + margin;
}
```

#### 2.2.3 智能拆分主函数
```javascript
/**
 * 智能拆分内容为多张图片
 * @param {string} markdownText - Markdown文本
 * @param {object} styleConfig - 样式配置
 * @returns {Array} 拆分后的图片数据数组
 */
function splitContentToImages(markdownText, styleConfig) {
    // 解析Markdown为HTML元素
    const htmlElements = parseMarkdownToElements(markdownText);
    
    // 提取文章结构
    const articleStructure = extractArticleStructure(markdownText);
    
    // 生成图片数据
    const images = [];
    
    // 1. 生成主图
    images.push(generateMainImage(articleStructure, styleConfig));
    
    // 2. 生成内容图（智能拆分）
    const contentImages = generateContentImages(
        htmlElements, 
        articleStructure, 
        styleConfig
    );
    images.push(...contentImages);
    
    // 3. 生成尾图
    images.push(generateEndImage(styleConfig));
    
    return images;
}

/**
 * 解析Markdown为HTML元素数组
 * @param {string} markdownText - Markdown文本
 * @returns {Array} HTML元素数组
 */
function parseMarkdownToElements(markdownText) {
    // 使用marked.js解析
    const html = marked.parse(markdownText);
    
    // 创建临时DOM解析
    const tempDiv = document.createElement('div');
    tempDiv.innerHTML = html;
    
    const elements = [];
    
    // 遍历子元素
    tempDiv.childNodes.forEach(node => {
        if (node.nodeType === Node.ELEMENT_NODE) {
            elements.push({
                type: node.tagName.toLowerCase(),
                content: node.innerHTML || node.textContent,
                attributes: getAttributes(node)
            });
        }
    });
    
    return elements;
}

/**
 * 提取文章结构
 * @param {string} markdownText - Markdown文本
 * @returns {object} 文章结构对象
 */
function extractArticleStructure(markdownText) {
    const lines = markdownText.split('\n').filter(line => line.trim());
    
    return {
        title: lines[0] || '标题',
        subtitle: lines[1] || '副标题',
        summary: lines[2] || '摘要',
        content: lines.slice(3).join('\n')
    };
}

/**
 * 生成主图数据
 * @param {object} articleStructure - 文章结构
 * @param {object} styleConfig - 样式配置
 * @returns {object} 主图数据
 */
function generateMainImage(articleStructure, styleConfig) {
    return {
        type: 'main',
        title: articleStructure.title,
        author: document.getElementById('author-input').value,
        showAuthor: document.getElementById('show-author').checked,
        followText: document.getElementById('follow-text-input').value,
        showFollow: document.getElementById('show-follow-main').checked,
        style: styleConfig
    };
}

/**
 * 生成内容图（智能拆分）
 * @param {Array} elements - HTML元素数组
 * @param {object} articleStructure - 文章结构
 * @param {object} styleConfig - 样式配置
 * @returns {Array} 内容图数组
 */
function generateContentImages(elements, articleStructure, styleConfig) {
    const images = [];
    let currentImage = {
        type: 'content',
        index: 1,
        subtitle: articleStructure.subtitle,
        elements: [],
        currentHeight: 0
    };
    
    const availableHeight = IMAGE_CONFIG.availableHeight;
    const maxWidth = IMAGE_CONFIG.width - IMAGE_CONFIG.padding * 2;
    
    // 添加页眉高度
    const showHeader = document.getElementById('show-follow-content-header').checked;
    if (showHeader) {
        currentImage.currentHeight += IMAGE_CONFIG.headerHeight;
    }
    
    // 添加副标题高度
    currentImage.currentHeight += IMAGE_CONFIG.subtitleHeight;
    
    // 添加页脚预留高度
    const showFooter = document.getElementById('show-follow-content-footer').checked;
    if (showFooter) {
        currentImage.currentHeight += IMAGE_CONFIG.footerHeight;
    }
    
    // 遍历元素，智能分配到图片
    for (let i = 0; i < elements.length; i++) {
        const element = elements[i];
        const elementHeight = calculateElementHeight(element, maxWidth, styleConfig);
        
        // 检查是否超出当前图片容量
        if (currentImage.currentHeight + elementHeight > availableHeight) {
            // 保存当前图片
            images.push({...currentImage});
            
            // 创建新图片
            currentImage = {
                type: 'content',
                index: images.length + 1,
                subtitle: articleStructure.subtitle,
                elements: [],
                currentHeight: IMAGE_CONFIG.headerHeight + IMAGE_CONFIG.subtitleHeight + IMAGE_CONFIG.footerHeight
            };
        }
        
        // 添加元素到当前图片
        currentImage.elements.push(element);
        currentImage.currentHeight += elementHeight;
    }
    
    // 添加最后一个图片
    if (currentImage.elements.length > 0) {
        images.push(currentImage);
    }
    
    return images;
}

/**
 * 计算元素高度
 * @param {object} element - 元素对象
 * @param {number} maxWidth - 最大宽度
 * @param {object} styleConfig - 样式配置
 * @returns {number} 元素高度
 */
function calculateElementHeight(element, maxWidth, styleConfig) {
    switch (element.type) {
        case 'h1':
            return HEADING_HEIGHT_MAP['h1'] + 20;
        case 'h2':
            return HEADING_HEIGHT_MAP['h2'] + 15;
        case 'h3':
            return HEADING_HEIGHT_MAP['h3'] + 15;
        case 'h4':
            return HEADING_HEIGHT_MAP['h4'] + 10;
        case 'h5':
            return HEADING_HEIGHT_MAP['h5'] + 10;
        case 'h6':
            return HEADING_HEIGHT_MAP['h6'] + 10;
        case 'p':
            return calculateParagraphHeight(
                element.content, 
                '16px', 
                maxWidth, 
                false, 
                styleConfig.background
            );
        case 'blockquote':
            return calculateQuoteHeight(
                element.content,
                '16px',
                maxWidth,
                true
            );
        case 'img':
            const width = element.attributes.width || maxWidth;
            const height = element.attributes.height || 600;
            return calculateImageHeight(width, height, maxWidth);
        case 'ul':
        case 'ol':
            return calculateListHeight(element.content, maxWidth);
        default:
            return 30;
    }
}

/**
 * 生成尾图数据
 * @param {object} styleConfig - 样式配置
 * @returns {object} 尾图数据
 */
function generateEndImage(styleConfig) {
    return {
        type: 'end',
        followText: document.getElementById('follow-text-input').value,
        showFollow: document.getElementById('show-follow-end').checked,
        style: styleConfig
    };
}
```

### 2.3 右侧预览区域改造

#### 2.3.1 HTML结构
```html
<div class="preview-box" id="preview-box">
    <div class="preview-header">
        <h3>📸 图片预览</h3>
        <span id="image-count">共 0 张</span>
    </div>
    
    <!-- 双列网格预览 -->
    <div class="preview-grid" id="preview-grid">
        <!-- 动态生成的图片卡片 -->
    </div>
</div>
```

#### 2.3.2 CSS样式
```css
/* 预览区域 */
.preview-box {
    flex: 1;
    display: flex;
    flex-direction: column;
    background: #f8f9fa;
    border-radius: 8px;
    overflow: hidden;
}

.preview-header {
    padding: 15px;
    background: white;
    border-bottom: 1px solid #e0e0e0;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.preview-header h3 {
    margin: 0;
    font-size: 16px;
    color: #333;
}

#image-count {
    font-size: 14px;
    color: #666;
}

/* 双列网格布局 */
.preview-grid {
    flex: 1;
    padding: 20px;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 20px;
    overflow-y: auto;
}

/* 图片卡片 */
.image-card {
    background: white;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    transition: transform 0.3s, box-shadow 0.3s;
    aspect-ratio: 3 / 4;
}

.image-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 8px 16px rgba(0,0,0,0.15);
}

.image-card-inner {
    width: 100%;
    height: 100%;
    position: relative;
    overflow: hidden;
}

.image-card-content {
    width: 100%;
    height: 100%;
    transform-origin: top left;
}

.image-card-label {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 8px 12px;
    background: rgba(0,0,0,0.7);
    color: white;
    font-size: 12px;
    text-align: center;
}

/* 批量保存按钮 */
.batch-save-btn {
    position: fixed;
    bottom: 30px;
    right: 30px;
    padding: 12px 24px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border: none;
    border-radius: 30px;
    cursor: pointer;
    font-size: 16px;
    font-weight: 600;
    box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
    transition: all 0.3s ease;
    z-index: 1000;
}

.batch-save-btn:hover {
    transform: translateY(-3px);
    box-shadow: 0 6px 16px rgba(102, 126, 234, 0.6);
}

.batch-save-btn:active {
    transform: translateY(-1px);
}

/* 小红书卡片样式 */
.xiaohongshu-card {
    width: 1080px;
    height: 1440px;
    padding: 60px;
    box-sizing: border-box;
    background: white;
    position: relative;
    overflow: hidden;
}

.xiaohongshu-card-header {
    text-align: center;
    margin-bottom: 40px;
    padding-bottom: 20px;
    border-bottom: 2px solid #f0f0f0;
}

.xiaohongshu-card-title {
    font-size: 64px;
    font-weight: 700;
    line-height: 1.2;
    margin-bottom: 20px;
}

.xiaohongshu-card-author {
    font-size: 32px;
    color: #666;
}

.xiaohongshu-card-content {
    flex: 1;
    overflow: hidden;
}

.xiaohongshu-card-footer {
    text-align: center;
    margin-top: 40px;
    padding-top: 20px;
    border-top: 2px solid #f0f0f0;
    font-size: 28px;
    color: #666;
}
```

#### 2.3.3 JavaScript渲染函数
```javascript
/**
 * 渲染预览网格
 * @param {Array} images - 图片数据数组
 */
function renderPreviewGrid(images) {
    const grid = document.getElementById('preview-grid');
    const countSpan = document.getElementById('image-count');
    
    // 清空现有内容
    grid.innerHTML = '';
    
    // 更新计数
    countSpan.textContent = `共 ${images.length} 张`;
    
    // 渲染每个图片卡片
    images.forEach((image, index) => {
        const card = createImageCard(image, index);
        grid.appendChild(card);
    });
}

/**
 * 创建图片卡片
 * @param {object} imageData - 图片数据
 * @param {number} index - 索引
 * @returns {HTMLElement} 图片卡片元素
 */
function createImageCard(imageData, index) {
    const card = document.createElement('div');
    card.className = 'image-card';
    
    const inner = document.createElement('div');
    inner.className = 'image-card-inner';
    
    const content = document.createElement('div');
    content.className = 'image-card-content';
    
    // 渲染小红书卡片内容
    const cardHTML = renderXiaohongshuCard(imageData);
    content.innerHTML = cardHTML;
    
    // 缩放显示（保持3:4比例）
    const scale = 0.25; // 1080x1440 → 270x360
    content.style.transform = `scale(${scale})`;
    content.style.transformOrigin = 'top left';
    content.style.width = `${1080 / scale}px`;
    content.style.height = `${1440 / scale}px`;
    
    inner.appendChild(content);
    card.appendChild(inner);
    
    // 添加标签
    const label = document.createElement('div');
    label.className = 'image-card-label';
    label.textContent = getCardLabel(imageData, index + 1);
    card.appendChild(label);
    
    return card;
}

/**
 * 获取卡片标签
 * @param {object} imageData - 图片数据
 * @param {number} number - 序号
 * @returns {string} 标签文本
 */
function getCardLabel(imageData, number) {
    switch (imageData.type) {
        case 'main':
            return `0${number} - 主图`;
        case 'content':
            return `0${number} - 内容图${imageData.index}`;
        case 'end':
            return `0${number} - 尾图`;
        default:
            return `0${number}`;
    }
}

/**
 * 渲染小红书卡片HTML
 * @param {object} imageData - 图片数据
 * @returns {string} HTML字符串
 */
function renderXiaohongshuCard(imageData) {
    const showFollow = imageData.showFollow || false;
    const followText = imageData.followText || '';
    const style = imageData.style || {};
    
    let html = `<div class="xiaohongshu-card" style="background: ${style.background || '#ffffff'};">`;
    
    // 页眉（关注语）
    if (showFollow && followText) {
        html += `
            <div class="xiaohongshu-card-header">
                <div style="font-size: 24px; color: #999;">${followText}</div>
            </div>
        `;
    }
    
    // 根据类型渲染内容
    switch (imageData.type) {
        case 'main':
            html += renderMainCardContent(imageData);
            break;
        case 'content':
            html += renderContentCardContent(imageData);
            break;
        case 'end':
            html += renderEndCardContent(imageData);
            break;
    }
    
    // 页脚（关注语）
    if (showFollow && followText) {
        html += `
            <div class="xiaohongshu-card-footer">
                <div>${followText}</div>
            </div>
        `;
    }
    
    html += '</div>';
    
    return html;
}

/**
 * 渲染主图内容
 * @param {object} imageData - 图片数据
 * @returns {string} HTML字符串
 */
function renderMainCardContent(imageData) {
    return `
        <div class="xiaohongshu-card-title" style="text-align: center;">
            ${imageData.title}
        </div>
        ${imageData.showAuthor ? `
            <div class="xiaohongshu-card-author" style="text-align: center;">
                @${imageData.author}
            </div>
        ` : ''}
    `;
}

/**
 * 渲染内容图内容
 * @param {object} imageData - 图片数据
 * @returns {string} HTML字符串
 */
function renderContentCardContent(imageData) {
    let html = `
        <div class="xiaohongshu-card-subtitle" style="text-align: center; font-size: 36px; margin-bottom: 30px;">
            ${imageData.subtitle}
        </div>
    `;
    
    // 渲染元素
    imageData.elements.forEach(element => {
        html += renderElement(element);
    });
    
    return html;
}

/**
 * 渲染尾图内容
 * @param {object} imageData - 图片数据
 * @returns {string} HTML字符串
 */
function renderEndCardContent(imageData) {
    return `
        <div style="text-align: center; padding: 100px 0;">
            <div style="font-size: 48px; margin-bottom: 30px;">✨</div>
            <div style="font-size: 32px; color: #666;">感谢阅读</div>
        </div>
    `;
}

/**
 * 渲染元素
 * @param {object} element - 元素对象
 * @returns {string} HTML字符串
 */
function renderElement(element) {
    switch (element.type) {
        case 'h1':
            return `<h1 style="font-size: 48px; margin: 30px 0;">${element.content}</h1>`;
        case 'h2':
            return `<h2 style="font-size: 40px; margin: 25px 0;">${element.content}</h2>`;
        case 'h3':
            return `<h3 style="font-size: 32px; margin: 20px 0;">${element.content}</h3>`;
        case 'p':
            return `<p style="font-size: 24px; line-height: 1.8; margin: 20px 0;">${element.content}</p>`;
        case 'blockquote':
            return `<blockquote style="font-size: 24px; padding: 20px; margin: 20px 0; background: #f5f5f5; border-left: 4px solid #667eea;">${element.content}</blockquote>`;
        case 'img':
            return `<img src="${element.attributes.src}" style="max-width: 100%; margin: 20px 0;" alt="">`;
        default:
            return `<div style="font-size: 24px; margin: 20px 0;">${element.content}</div>`;
    }
}
```

### 2.4 批量保存功能

#### 2.4.1 批量保存按钮
```html
<button class="batch-save-btn" onclick="batchSaveImages()">
    📥 批量保存
</button>
```

#### 2.4.2 批量保存函数
```javascript
/**
 * 批量保存图片
 */
async function batchSaveImages() {
    const images = getCurrentImages();
    
    if (images.length === 0) {
        showToast('没有可保存的图片');
        return;
    }
    
    showToast(`正在生成 ${images.length} 张图片...`);
    
    try {
        // 引入html2canvas（如果未加载）
        if (typeof html2canvas === 'undefined') {
            await loadHtml2Canvas();
        }
        
        // 逐个生成并下载图片
        for (let i = 0; i < images.length; i++) {
            const imageData = images[i];
            const filename = getFilename(imageData, i + 1);
            
            await generateAndSaveImage(imageData, filename);
            
            showToast(`已保存 ${i + 1}/${images.length}: ${filename}`);
        }
        
        showToast('✅ 所有图片保存完成！');
    } catch (error) {
        console.error('批量保存失败:', error);
        showToast('❌ 保存失败，请重试');
    }
}

/**
 * 加载html2canvas库
 */
async function loadHtml2Canvas() {
    return new Promise((resolve, reject) => {
        const script = document.createElement('script');
        script.src = 'https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js';
        script.onload = resolve;
        script.onerror = reject;
        document.head.appendChild(script);
    });
}

/**
 * 获取当前图片数据
 * @returns {Array} 图片数据数组
 */
function getCurrentImages() {
    // 从全局变量或缓存中获取
    return window.currentImages || [];
}

/**
 * 获取文件名
 * @param {object} imageData - 图片数据
 * @param {number} index - 序号
 * @returns {string} 文件名
 */
function getFilename(imageData, index) {
    const prefix = index < 10 ? `0${index}` : index;
    
    switch (imageData.type) {
        case 'main':
            return `${prefix}-主图.png`;
        case 'content':
            return `${prefix}-内容图${imageData.index}.png`;
        case 'end':
            return `${prefix}-尾图.png`;
        default:
            return `${prefix}.png`;
    }
}

/**
 * 生成并保存单张图片
 * @param {object} imageData - 图片数据
 * @param {string} filename - 文件名
 */
async function generateAndSaveImage(imageData, filename) {
    // 创建临时容器
    const container = document.createElement('div');
    container.style.position = 'fixed';
    container.style.left = '-9999px';
    container.style.top = '0';
    container.style.width = '1080px';
    container.style.height = '1440px';
    container.style.background = imageData.style.background || '#ffffff';
    
    // 渲染卡片内容
    container.innerHTML = renderXiaohongshuCard(imageData);
    
    document.body.appendChild(container);
    
    try {
        // 使用html2canvas生成图片
        const canvas = await html2canvas(container, {
            width: 1080,
            height: 1440,
            scale: 2, // 高清输出
            useCORS: true,
            allowTaint: false,
            backgroundColor: imageData.style.background || '#ffffff'
        });
        
        // 转换为Blob
        const blob = await new Promise(resolve => canvas.toBlob(resolve, 'image/png'));
        
        // 下载文件
        downloadBlob(blob, filename);
    } finally {
        // 清理临时容器
        document.body.removeChild(container);
    }
}

/**
 * 下载Blob文件
 * @param {Blob} blob - Blob对象
 * @param {string} filename - 文件名
 */
function downloadBlob(blob, filename) {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
}

/**
 * 显示提示消息
 * @param {string} message - 消息内容
 */
function showToast(message) {
    const toast = document.createElement('div');
    toast.className = 'toast';
    toast.textContent = message;
    toast.style.position = 'fixed';
    toast.style.top = '20px';
    toast.style.left = '50%';
    toast.style.transform = 'translateX(-50%)';
    toast.style.background = 'rgba(0,0,0,0.8)';
    toast.style.color = 'white';
    toast.style.padding = '12px 24px';
    toast.style.borderRadius = '20px';
    toast.style.zIndex = '10000';
    toast.style.opacity = '0';
    toast.style.transition = 'opacity 0.3s';
    
    document.body.appendChild(toast);
    
    setTimeout(() => toast.style.opacity = '1', 10);
    setTimeout(() => {
        toast.style.opacity = '0';
        setTimeout(() => document.body.removeChild(toast), 300);
    }, 2000);
}
```

### 2.5 主题切换功能

#### 2.5.1 主题切换函数
```javascript
let currentThemeIndex = 0;
const themeKeys = Object.keys(themes);

/**
 * 应用主题
 * @param {string} themeKey - 主题键
 */
function applyTheme(themeKey) {
    const theme = themes[themeKey];
    if (!theme) return;
    
    // 更新样式选择器
    document.getElementById('style-select').value = theme.h1Style;
    document.getElementById('h2-style-select').value = theme.h2Style;
    document.getElementById('heading-style-select').value = theme.h3h6Style;
    document.getElementById('template-style-select').value = theme.font;
    document.getElementById('style-block-background').value = theme.background;
    document.getElementById('quote-style-select').value = theme.quoteStyle;
    document.getElementById('quote-symbol-select').value = theme.quoteSymbol;
    
    // 触发样式更新
    changeHtml(themeKey);
    
    // 更新主题名称显示
    document.getElementById('current-theme-name').textContent = theme.name;
    
    // 重新生成预览
    updatePreview();
}

/**
 * 上一个主题
 */
function prevTheme() {
    currentThemeIndex = (currentThemeIndex - 1 + themeKeys.length) % themeKeys.length;
    const themeKey = themeKeys[currentThemeIndex];
    document.getElementById('theme-select').value = themeKey;
    applyTheme(themeKey);
}

/**
 * 下一个主题
 */
function nextTheme() {
    currentThemeIndex = (currentThemeIndex + 1) % themeKeys.length;
    const themeKey = themeKeys[currentThemeIndex];
    document.getElementById('theme-select').value = themeKey;
    applyTheme(themeKey);
}

// 绑定事件
document.getElementById('theme-prev-btn').addEventListener('click', prevTheme);
document.getElementById('theme-next-btn').addEventListener('click', nextTheme);
```

### 2.6 实时更新机制

#### 2.6.1 监听所有变化
```javascript
/**
 * 初始化事件监听
 */
function initEventListeners() {
    // MD内容输入
    document.getElementById('editor').addEventListener('input', debounce(updatePreview, 500));
    
    // 作者和关注语
    document.getElementById('author-input').addEventListener('input', updatePreview);
    document.getElementById('follow-text-input').addEventListener('input', updatePreview);
    
    // 显示控制复选框
    const checkboxes = [
        'show-author',
        'show-follow-main',
        'show-follow-content-header',
        'show-follow-content-footer',
        'show-follow-end'
    ];
    
    checkboxes.forEach(id => {
        document.getElementById(id).addEventListener('change', updatePreview);
    });
    
    // 样式选择器变化
    const styleSelectors = [
        'style-select',
        'h2-style-select',
        'heading-style-select',
        'template-style-select',
        'style-block-background',
        'quote-style-select',
        'quote-symbol-select',
        'quote-font-select',
        'quote-font-size-select'
    ];
    
    styleSelectors.forEach(id => {
        const element = document.getElementById(id);
        if (element) {
            element.addEventListener('change', debounce(updatePreview, 300));
        }
    });
}

/**
 * 防抖函数
 * @param {Function} func - 要防抖的函数
 * @param {number} wait - 等待时间
 * @returns {Function} 防抖后的函数
 */
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

/**
 * 更新预览
 */
function updatePreview() {
    const markdownText = document.getElementById('editor').value;
    const styleConfig = getCurrentStyleConfig();
    
    // 拆分内容为图片
    const images = splitContentToImages(markdownText, styleConfig);
    
    // 保存到全局变量
    window.currentImages = images;
    
    // 渲染预览
    renderPreviewGrid(images);
}

/**
 * 获取当前样式配置
 * @returns {object} 样式配置对象
 */
function getCurrentStyleConfig() {
    return {
        h1Style: document.getElementById('style-select').value,
        h2Style: document.getElementById('h2-style-select').value,
        h3h6Style: document.getElementById('heading-style-select').value,
        font: document.getElementById('template-style-select').value,
        background: document.getElementById('style-block-background').value,
        quoteStyle: document.getElementById('quote-style-select').value,
        quoteSymbol: document.getElementById('quote-symbol-select').value,
        quoteFont: document.getElementById('quote-font-select').value,
        quoteFontSize: document.getElementById('quote-font-size-select').value
    };
}
```

## 三、实现步骤

### 步骤1：HTML结构调整
1. 在样式设置区域最上方新增"小红书设置区域"
2. 修改右侧预览区域，从单个预览改为双列网格
3. 添加批量保存按钮（右下角悬停）

### 步骤2：CSS样式添加
1. 添加小红书设置区域样式
2. 添加双列网格布局样式
3. 添加图片卡片样式
4. 添加批量保存按钮样式
5. 添加小红书卡片基础样式

### 步骤3：JavaScript核心功能实现
1. 实现主题配置和切换功能
2. 实现智能拆分算法（基于像素高度）
3. 实现图片生成和渲染功能
4. 实现批量保存功能
5. 实现实时更新机制

### 步骤4：集成现有功能
1. 保持所有现有样式设置不变
2. 保持图片上传功能不变
3. 保持Markdown解析功能不变

### 步骤5：测试和优化
1. 测试不同主题切换
2. 测试智能拆分准确性
3. 测试批量保存功能
4. 测试实时更新性能
5. 优化用户体验

## 四、技术要点

### 4.1 核心算法
- **像素高度计算**：基于字体、行高、元素类型精确计算
- **智能拆分**：确保内容在合适的边界分割
- **性能优化**：防抖处理，避免频繁重渲染

### 4.2 关键技术
- **html2canvas**：生成高清图片
- **CSS Grid**：双列网格布局
- **事件监听**：实时响应变化
- **防抖优化**：提升性能

### 4.3 兼容性
- 保持现有100%兼容
- 不影响任何现有功能
- 平滑升级

## 五、注意事项

### 5.1 性能优化
- 预览时使用缩略图（0.25倍）
- 保存时才生成高清图（2倍）
- 防抖处理避免频繁计算

### 5.2 用户体验
- 实时预览，所见即所得
- 清晰的进度提示
- 友好的错误处理

### 5.3 代码质量
- 模块化设计
- 注释完善
- 易于维护和扩展

## 六、预期效果

### 6.1 功能完整性
✅ 5种小红书主题风格
✅ 智能内容拆分
✅ 批量图片生成和保存
✅ 右侧双列网格预览
✅ 作者和关注语自定义
✅ 保持所有现有功能

### 6.2 用户体验
✅ 简洁直观的界面
✅ 流畅的主题切换
✅ 实时的预览更新
✅ 高效的批量保存

### 6.3 技术指标
✅ 图片尺寸：1080x1440（3:4）
✅ 输出格式：PNG高清
✅ 拆分准确率：>95%
✅ 渲染速度：<2秒

---

*文档创建时间：2026年2月28日*
*预计开发时间：2-3天*
*技术栈：HTML5 + CSS3 + JavaScript (ES6+)*