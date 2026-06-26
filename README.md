# bili-link-cleaner-script
👇AI油猴脚本
```javascript
// ==UserScript==
// @name         B站链接去跟踪参数
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  移除首页跳转、视频页、空间页链接参数
// @author       FuckBiliLink
// @match        *://*.bilibili.com/*
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function() {
    'use strict';

    // 视频页需要移除的黑名单参数
    const BLOCK_PARAMS = ['spm_id_from',
                          'vd_source',
                          'from_source',
                          'from',
                          'msource',
                          'bsource',
                          'spmid'];

    // 移除URL中的参数
    function cleanUrl(rawUrl) {
        if (!rawUrl) return rawUrl;
        try {
            const url = new URL(rawUrl, location.origin);
            if (url.pathname.startsWith('/video/') && url.search) {
                url.search = '';
                return url.toString();
            }
        } catch (e) {}
        return rawUrl;
    }

    // 移除当前地址栏参数 (针对视频页应用黑名单)
    function cleanCurrentUrl() {
        try {
            const url = new URL(location.href);
            if (url.pathname.startsWith('/video/') && url.search) {
                let hasChange = false;
                BLOCK_PARAMS.forEach(param => {
                    if (url.searchParams.has(param)) {
                        url.searchParams.delete(param);
                        hasChange = true;
                    }
                });
                if (hasChange) {
                    history.replaceState(null, null, url.toString());
                }
            }
        } catch (e) {}
    }

    // 劫持History API
    const rewriteHistory = (type) => {
        const original = history[type];
        return function(...args) {
            if (args[2]) args[2] = cleanUrl(args[2]);
            const result = original.apply(this, args);
            cleanCurrentUrl();
            return result;
        };
    };
    history.pushState = rewriteHistory('pushState');
    history.replaceState = rewriteHistory('replaceState');

    // 监听路由与地址栏变更
    window.addEventListener('popstate', cleanCurrentUrl, true);
    window.addEventListener('hashchange', cleanCurrentUrl, true);

    // 页面加载生命周期介入
    cleanCurrentUrl();
    document.addEventListener('DOMContentLoaded', cleanCurrentUrl);
    window.addEventListener('load', cleanCurrentUrl);

    // 净化单个 <a> 标签并克隆以解绑已有事件
    function sanitizeAnchor(anchor) {
        if (!anchor.href || anchor.dataset.cleaned) return;

        try {
            const url = new URL(anchor.href);
            if (url.hostname.includes('bilibili.com') && url.pathname.startsWith('/video/')) {
                url.search = '';

                const clonedAnchor = anchor.cloneNode(true);
                clonedAnchor.href = url.toString();
                clonedAnchor.dataset.cleaned = 'true';
                clonedAnchor.rel = 'noreferrer';

                clonedAnchor.removeAttribute('data-spmid');
                clonedAnchor.removeAttribute('data-mod');
                clonedAnchor.removeAttribute('data-idx');

                if (anchor.parentNode) {
                    anchor.parentNode.replaceChild(clonedAnchor, anchor);
                }
            }
        } catch (e) {}
    }

    // 监听动态添加的视频卡片
    const observer = new MutationObserver((mutations) => {
        for (const mutation of mutations) {
            if (mutation.addedNodes.length) {
                const anchors = document.querySelectorAll('a.bili-video-card__image--link, .bili-video-card__info--tit a');
                anchors.forEach(sanitizeAnchor);
            }
        }
    });

    window.addEventListener('DOMContentLoaded', () => {
        const anchors = document.querySelectorAll('a.bili-video-card__image--link, .bili-video-card__info--tit a');
        anchors.forEach(sanitizeAnchor);
        observer.observe(document.body, { childList: true, subtree: true });
    });
})();
```
