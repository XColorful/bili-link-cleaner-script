# bili-link-cleaner-script
👇AI油猴脚本
```javascript
// ==UserScript==
// @name         B站链接去跟踪参数
// @namespace    http://tampermonkey.net/
// @version      1.1
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
                          'spmid',
                          'trackid'];

    // 移除URL中的参数
    function cleanUrl(rawUrl) {
        if (!rawUrl) return rawUrl;
        try {
            const url = new URL(rawUrl, location.origin);
            if (url.hostname.includes('bilibili.com')) {
                if (url.pathname.startsWith('/video/')) {
                    let hasBlockParam = BLOCK_PARAMS.some(param => url.searchParams.has(param));
                    if (hasBlockParam) {
                        BLOCK_PARAMS.forEach(param => url.searchParams.delete(param));
                        if (!url.searchParams.toString()) {
                            url.search = '';
                        }
                        return url.toString();
                    }
                } else if (url.hostname.startsWith('space.')) {
                    let hasBlockParam = BLOCK_PARAMS.some(param => url.searchParams.has(param));
                    if (hasBlockParam) {
                        BLOCK_PARAMS.forEach(param => url.searchParams.delete(param));
                        return url.toString();
                    }
                }
            }
        } catch (e) {}
        return rawUrl;
    }

    // 拦截 <a> 标签的 href 属性赋值行为，彻底解决异步框架回写问题
    try {
        const originalHrefGet = Object.getOwnPropertyDescriptor(HTMLAnchorElement.prototype, 'href').get;
        const originalHrefSet = Object.getOwnPropertyDescriptor(HTMLAnchorElement.prototype, 'href').set;

        Object.defineProperty(HTMLAnchorElement.prototype, 'href', {
            get: function() {
                return originalHrefGet.call(this);
            },
            set: function(val) {
                const cleaned = cleanUrl(val);
                originalHrefSet.call(this, cleaned);
            },
            configurable: true,
            enumerable: true
        });
    } catch (e) {}

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

    // 净化单个 <a> 标签
    function sanitizeAnchor(anchor) {
        if (!anchor.href) return;

        try {
            const currentHref = anchor.href;
            const cleanedHref = cleanUrl(currentHref);

            if (currentHref !== cleanedHref) {
                anchor.href = cleanedHref;
                anchor.rel = 'noreferrer';
                anchor.removeAttribute('data-spmid');
                anchor.removeAttribute('data-mod');
                anchor.removeAttribute('data-idx');
            }
        } catch (e) {}
    }

    // 处理选定范围内的所有a标签
    function processAnchors(container) {
        const root = container || document;
        const anchors = root.querySelectorAll('a[href*="/video/"], a[href*="space.bilibili.com"]');
        anchors.forEach(sanitizeAnchor);
    }

    // 监听动态添加的视频卡片与UP主主页链接
    const observer = new MutationObserver((mutations) => {
        for (const mutation of mutations) {
            if (mutation.addedNodes.length) {
                mutation.addedNodes.forEach(node => {
                    if (node.nodeType === Node.ELEMENT_NODE) {
                        processAnchors(node);
                    }
                });
            } else if (mutation.type === 'attributes' && mutation.attributeName === 'href') {
                sanitizeAnchor(mutation.target);
            }
        }
    });

    window.addEventListener('DOMContentLoaded', () => {
        processAnchors(document.body);
        observer.observe(document.body, {
            childList: true,
            subtree: true,
            attributes: true,
            attributeFilter: ['href']
        });
    });
})();
```
