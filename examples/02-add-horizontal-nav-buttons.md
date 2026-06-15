# 横向滑动导航按钮（grid-nav）模板

把一个横排卡片区从「只能用鼠标滚轮滑」升级为「左右圆形箭头按钮 + 滚轮」。

## 适用场景

任何用 `<div class="other-grid">` 横向 flex 布局的区域：
- 「其他项目」(`#other-projects`)
- 「个人项目」(`#personal-projects`)
- 自己加的任何横排卡片区

## 改 HTML

把原本的 `<div class="other-grid">` 包一层 `<div class="other-grid-frame">`，frame 内加两个按钮。

**Before**：
```html
<section id="personal-projects" class="section-reveal">
  <div class="container">
    <div class="section-head text-center">...</div>

    <div class="other-grid">
      <a class="bento-card other-card ...">...</a>
      <a class="bento-card other-card ...">...</a>
    </div>
  </div>
</section>
```

**After**：
```html
<section id="personal-projects" class="section-reveal">
  <div class="container">
    <div class="section-head text-center">...</div>

    <div class="other-grid-frame">
      <button class="grid-nav grid-nav-prev" aria-label="上一张项目卡片" type="button">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none"
             stroke="currentColor" stroke-width="2.5"
             stroke-linecap="round" stroke-linejoin="round">
          <polyline points="15 18 9 12 15 6"/>
        </svg>
      </button>
      <button class="grid-nav grid-nav-next" aria-label="下一张项目卡片" type="button">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none"
             stroke="currentColor" stroke-width="2.5"
             stroke-linecap="round" stroke-linejoin="round">
          <polyline points="9 18 15 12 9 6"/>
        </svg>
      </button>
      <div class="other-grid">
        <a class="bento-card other-card ...">...</a>
        <a class="bento-card other-card ...">...</a>
      </div>
    </div>
  </div>
</section>
```

## 改 JS（关键）

如果项目里已经有「其他项目」的横滑实现，**最容易漏的一步**：
原 JS 用 `document.querySelector('.other-grid')` 只绑了第一个 grid。
改成多 grid 时必须用 `querySelectorAll('.other-grid-frame').forEach()`。

```js
function setupHorizontalScroll() {
  // 遍历所有 .other-grid-frame，让所有横排区都生效
  document.querySelectorAll('.other-grid-frame').forEach((frame) => {
    const grid = frame.querySelector('.other-grid');
    if (!grid) return;

    const STEP = 404;             // 卡片宽 380 + gap 24 = 单次滚动 404
    const SCROLL_MULTIPLIER = 2.4; // 滚轮灵敏度

    // 鼠标滚轮 → 横滑
    grid.addEventListener('wheel', (e) => {
      if (e.deltaY === 0 || Math.abs(e.deltaX) > Math.abs(e.deltaY)) return;
      const atStart = grid.scrollLeft <= 0;
      const atEnd = grid.scrollLeft + grid.clientWidth >= grid.scrollWidth - 1;
      if ((e.deltaY > 0 && atEnd) || (e.deltaY < 0 && atStart)) return;
      e.preventDefault();
      grid.scrollLeft += e.deltaY * SCROLL_MULTIPLIER;
    }, { passive: false });

    // 左右按钮（每个 frame 内独立找）
    const prevBtn = frame.querySelector('.grid-nav-prev');
    const nextBtn = frame.querySelector('.grid-nav-next');
    if (prevBtn && nextBtn) {
      prevBtn.addEventListener('click', () => {
        grid.scrollBy({ left: -STEP, behavior: 'smooth' });
      });
      nextBtn.addEventListener('click', () => {
        grid.scrollBy({ left: STEP, behavior: 'smooth' });
      });

      // 到边自动隐藏对应按钮
      const updateBtns = () => {
        const atStart = grid.scrollLeft <= 2;
        const atEnd = grid.scrollLeft + grid.clientWidth >= grid.scrollWidth - 2;
        prevBtn.disabled = atStart;
        nextBtn.disabled = atEnd;
      };
      grid.addEventListener('scroll', updateBtns, { passive: true });
      window.addEventListener('resize', updateBtns);
      updateBtns();
      setTimeout(updateBtns, 100); // 等图片加载完再校准一次
    }
  });
}
```

## CSS 不用改

`.other-grid-frame` / `.grid-nav` / `.grid-nav-prev` / `.grid-nav-next` 这些类应该已经在 style.css 里定义过（圆形按钮 + 绝对定位 + 边界淡出动画）。

如果你的项目里没这些 CSS，参考：

```css
.other-grid-frame { position: relative; margin-top: 48px; }
.grid-nav {
  position: absolute; top: 50%; transform: translateY(-50%);
  width: 48px; height: 48px; border-radius: 50%;
  background: var(--bg-card); border: 1px solid var(--border);
  cursor: pointer; z-index: 10;
  display: flex; align-items: center; justify-content: center;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  transition: all 0.2s cubic-bezier(.34, 1.56, .64, 1);
}
.grid-nav:hover  { background: var(--primary); color: white; transform: translateY(-50%) scale(1.08); }
.grid-nav:disabled { opacity: 0; pointer-events: none; transform: translateY(-50%) scale(0.85); }
.grid-nav-prev { left: -64px; }
.grid-nav-next { right: -64px; }
@media (max-width: 1240px) {
  .grid-nav-prev { left: -8px; }
  .grid-nav-next { right: -8px; }
}
```

## 验证

改完后测试：
- 滚轮在任一横排区都能横滑（不只是第一个）
- 左右按钮点击有平滑滚动
- 到最左 → prev 按钮淡出消失；到最右 → next 淡出消失
- resize 窗口后按钮状态会自动更新
