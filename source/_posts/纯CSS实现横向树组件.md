---
title: 纯CSS实现横向树组件
toc: true
date: 2022-04-02 00:00:00
tags: [代码片段]
categories: [前端技术, 代码片段]
---
## CSS

```scss
//HorizontalTree.module.scss
$connect-line: 30px;
$child-padding-left: 10px;

.horizontal-tree ul {
  display: flex;
  flex-direction: column;
  position: relative;
  list-style-type: none;
  margin: 0;
  padding: 2px 0 2px $connect-line;
}

.horizontal-tree ul ul::before {
  content: '';
  border-top: 2px solid cyan;
  position: absolute;
  bottom: 0;
  left: 0;
  width: $connect-line;
  height: 50%;
}

.horizontal-tree li {
  display: flex;
  padding: 2px 0 2px $connect-line+$child-padding-left;
  position: relative;
  align-items: center;
}

.horizontal-tree li::before {
  content: '';
  border-left: 2px solid cyan;
  border-bottom: 2px solid cyan;
  position: absolute;
  left: 0;
  bottom: 50%;
  width: $connect-line;
  height: 50%;
}

.horizontal-tree li::after {
  content: '';
  border-left: 2px solid cyan;
  position: absolute;
  left: 0;
  bottom: 0;
  width: $connect-line;
  height: 50%;
}

.horizontal-tree li:first-child::before,
.horizontal-tree li:last-child::after {
  border-left: none;
}

.horizontal-tree li:last-child::before {
  border-bottom-left-radius: 4px;
}

.horizontal-tree li:first-child::before {
  border-bottom: none;
}

.horizontal-tree li:not(.horizontal-tree-root):first-child::after {
  border-top: 2px solid cyan;
  border-top-left-radius: 4px;
}
```

## 模板

```tsx
<div>
  <div class="horizontal-tree">
    <ul>
      <li class="horizontal-tree-root">
        父节点
        <ul>
          <li>
            123
          </li>
          <li>456
            <ul>
              <li>
                114
              </li>
              <li>514</li>
            </ul>
          </li>
        </ul>
      </li>
    </ul>
  </div>
</div>
```

