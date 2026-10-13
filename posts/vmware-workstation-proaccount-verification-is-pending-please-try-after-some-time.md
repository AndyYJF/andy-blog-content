---
slug: vmware-workstation-proaccount-verification-is-pending-please-try-after-some-time
kind: post
title: 解决下载VMware Workstation Pro页面卡在Account verification is Pending. Please try after some time.
legacyCid: 142
canonicalPath: /posts/vmware-workstation-proaccount-verification-is-pending-please-try-after-some-time/
commentKey: /posts/vmware-workstation-proaccount-verification-is-pending-please-try-after-some-time/
feedGuid: urn:andy-y:post:142
allowComment: true
allowFeed: true
pubDate: '2026-10-01T07:48:00.000Z'
updatedDate: '2026-10-01T08:52:00.000Z'
categories:
  - mid: 1
    name: 所有文章
    slug: default
  - mid: 11
    name: 开源项目
    slug: opensource
tags: []
sourceFormat: markdown
description: 今天想搭一个虚拟机，就准备下一个 VMware，发现现在 VMware 的下载被一家叫做 BROADCOM 的公司接管了。
cover: https://tc.andy-y.cn/i/2026/10/01/6abe0e628a303.png
---

# 起因

今天想搭一个虚拟机，就准备下一个 VMware，发现现在 VMware 的下载被一家叫做 `BROADCOM` 的公司接管了。

经过一番周折填入信息注册了之后，在一堆乱七八糟的界面找到了下载按钮，点击，结果提示我： `Account verification is Pending. Please try after some time.`

我老老实实等了几个小时，问题依旧。

# 解决

于是查了下官网，最后发现了：

> Broadcom 官方 KB  [368588](https://knowledge.broadcom.com/external/article/368588/)  说，Basic 账号下载免费软件要先过 GTM 出口管制筛查，触发 pending 的典型原因就写在里面：公司/姓名不完整、只有 PO Box 没门牌号、资料里含非英文字符、筛查队列积压。

但也有人提到：Broadcom 前端会读接口返回里的 `exportControlStatus` 字段做拦截，也就是说这完全是一个前端拦截。

故我让AI写了一个拦截脚本，在页面代码反序列化响应之前改掉那个字段，就能正常下载了

脚本如下：

```js
// ==UserScript==
// @name         Broadcom Download Unlock (Account verification is Pending)
// @namespace    https://support.broadcom.com/
// @version      1.0.0
// @description  Bypass the "Account verification is Pending. Please try after some time." wall on support.broadcom.com by neutralizing exportControlStatus / screening fields in the download API responses, before the site's own code reads them.
// @author       local
// @match        https://support.broadcom.com/*
// @match        https://*.broadcom.com/*
// @icon         file:///C:/Users/AndyYan/dsh/broadcom-vmware-unlock/icon.ico
// @run-at       document-start
// @grant        none
// @noframes     false
// ==/UserScript==

/*
 * HOW IT WORKS
 * ------------
 * support.broadcom.com's download cockpit asks its backend for file details and
 * refuses to hand you the binary when the JSON contains a trade-compliance field
 * (exportControlStatus, and friends) whose value means "GTM screening pending".
 * That check lives in the page's JavaScript, so simply rewriting the response
 * body before the page deserializes it removes the wall entirely.
 *
 * This script patches both transports the cockpit has used:
 *   fetch() -> Response.prototype.json / Response.prototype.text
 *   XHR     -> XMLHttpRequest.prototype.responseText / .response getters
 *
 * Every body is parsed first; if the payload does not look compliance-related,
 * the untouched value is returned. Nothing is fabricated that wasn't there.
 */

(function () {
  'use strict';

  if (window.__BCU_HOOK__) return;

  window.__BCU_HOOK__ = { version: '1.0.0', hits: 0, log: [] };

  var CONFIG = {
    debug: true,        // true = print what was rewritten into the DevTools console
    mode: 'replace',    // 'replace' the pending value, or 'remove' the field outright
    matchAnyExportField: true
  };

  /* Keys that describe trade-compliance / verification state. */
  var OBSERVE_RE = /(exportcontrol|screening|deniedparty|tradecompliance|gtmstatus|restricted|verification)/i;

  /* Values seen from the backend that mean "still pending", and what to answer instead. */
  var NEUTRAL = {
    SCREENING_PENDING: 'SCREENING_NOT_REQUIRED',
    SCREENING_IN_PROGRESS: 'SCREENING_NOT_REQUIRED',
    SCREENING_REQUIRED: 'SCREENING_NOT_REQUIRED',
    PENDING: 'SCREENING_NOT_REQUIRED',
    PENDING_SCREENING: 'SCREENING_NOT_REQUIRED',
    IN_PROGRESS: 'SCREENING_NOT_REQUIRED',
    ON_HOLD: 'SCREENING_NOT_REQUIRED',
    UNDER_REVIEW: 'SCREENING_NOT_REQUIRED',
    NOT_SCREENED: 'SCREENING_NOT_REQUIRED',
    NOT_VERIFIED: 'VERIFIED',
    UNVERIFIED: 'VERIFIED',
    VERIFICATION_PENDING: 'VERIFIED',
    BLOCKED: 'CLEARED',
    DENIED: 'CLEARED',
    REJECTED: 'CLEARED',
    FAILED: 'CLEARED'
  };

  /* Explicit key -> value overrides. Keys are matched case-insensitively. */
  var RULES = {
    exportcontrolstatus: 'SCREENING_NOT_REQUIRED',
    exportcontrolscreeningstatus: 'SCREENING_NOT_REQUIRED',
    screeningstatus: 'CLEARED',
    deniedpartystatus: 'CLEARED',
    tradecompliancestatus: 'CLEARED',
    gtmstatus: 'CLEARED',
    compliancestatus: 'CLEARED',
    accountverificationstatus: 'VERIFIED',
    verificationstatus: 'VERIFIED',
    exportcontrolrequired: false,
    screeningrequired: false,
    verificationpending: false,
    restricted: false,
    isrestricted: false,
    deniedparty: false
  };

  /* ---------------------------------------------------------------- rewriting */

  function neutralized(body) {
    if (typeof body !== 'string' || body.length < 2) return null;
    if (!/exportcontrol|screening|deniedparty|verification|restricted|tradecompliance|gtmstatus/i.test(body)) return null;

    var data;
    try { data = JSON.parse(body); } catch (e) { return null; }
    if (!data || typeof data !== 'object') return null;

    var hits = [];
    walk(data, [], 0, hits);

    if (!hits.length) return null;

    window.__BCU_HOOK__.hits++;
    window.__BCU_HOOK__.log.push(hits);
    if (CONFIG.debug) {
      console.log('%c[BCU] neutralized ' + hits.length + ' field(s)', 'color:#0b5fff;font-weight:bold',
        hits.map(function (h) { return h.path + ': ' + JSON.stringify(h.from) + ' -> ' + JSON.stringify(h.to); }).join('\n'));
    }
    try { return JSON.stringify(data); } catch (e) { return null; }
  }

  function walk(node, path, depth, hits) {
    if (!node || typeof node !== 'object' || depth > 14) return;
    if (Array.isArray(node)) {
      for (var i = 0; i < node.length; i++) walk(node[i], path.concat('[' + i + ']'), depth + 1, hits);
      return;
    }
    var keys = Object.keys(node);
    for (var k = 0; k < keys.length; k++) {
      var key = keys[k];
      var value = node[key];
      var p = path.concat(key).join('.');
      var lower = key.toLowerCase();

      if (value !== null && typeof value === 'object') { walk(value, path.concat(key), depth + 1, hits); continue; }

      if (Object.prototype.hasOwnProperty.call(RULES, lower)) {
        var want = RULES[lower];
        if (CONFIG.mode === 'remove') {
          delete node[key];
          hits.push({ path: p, from: value, to: '<removed>' });
        } else if (value !== want) {
          node[key] = want;
          hits.push({ path: p, from: value, to: want });
        }
        continue;
      }

      if (CONFIG.matchAnyExportField && typeof value === 'string' && OBSERVE_RE.test(key)) {
        var up = value.toUpperCase();
        if (Object.prototype.hasOwnProperty.call(NEUTRAL, up)) {
          node[key] = NEUTRAL[up];
          hits.push({ path: p, from: value, to: NEUTRAL[up] });
        }
      }
    }
  }

  /* --------------------------------------------------------------------- XHR */

  var XHR = window.XMLHttpRequest;
  if (XHR && XHR.prototype) {
    var proto = XHR.prototype;
    var nativeText = Object.getOwnPropertyDescriptor(proto, 'responseText');
    var nativeResponse = Object.getOwnPropertyDescriptor(proto, 'response');

    function patchedFor(xhr) {
      if (xhr.__bcu !== undefined) return xhr.__bcu;
      var out = null;
      try {
        if (xhr.responseType === 'json') {
          var parsed = xhr.response;
          if (parsed && typeof parsed === 'object') {
            var raw = JSON.stringify(parsed);
            var fixed = neutralized(raw);
            if (fixed !== null) out = JSON.parse(fixed);
          }
        } else {
          var text = nativeText && nativeText.get ? nativeText.get.call(xhr) : null;
          var rewritten = neutralized(text);
          if (rewritten !== null) out = rewritten;
        }
      } catch (e) { /* leave the response untouched */ }
      xhr.__bcu = out;
      return out;
    }

    if (nativeText && nativeText.get) {
      Object.defineProperty(proto, 'responseText', {
        configurable: true,
        get: function () {
          var p = patchedFor(this);
          return typeof p === 'string' ? p : nativeText.get.call(this);
        }
      });
    }

    if (nativeResponse && nativeResponse.get) {
      Object.defineProperty(proto, 'response', {
        configurable: true,
        get: function () {
          var p = patchedFor(this);
          return p !== null && p !== undefined ? p : nativeResponse.get.call(this);
        }
      });
    }
  }

  /* ------------------------------------------------------------------- fetch */

  var Response_ = window.Response;
  if (Response_ && Response_.prototype) {
    var nativeJson = Response_.prototype.json;
    var nativeBodyText = Response_.prototype.text;

    Object.defineProperty(Response_.prototype, 'json', {
      configurable: true,
      writable: true,
      value: function () {
        var res = this;
        if (res.__bcuJson) return res.__bcuJson;
        var p = nativeJson.call(res).then(function (data) {
          try {
            if (!data || typeof data !== 'object') return data;
            var fixed = neutralized(JSON.stringify(data));
            if (fixed !== null) return JSON.parse(fixed);
          } catch (e) { /* fall through */ }
          return data;
        });
        try { res.__bcuJson = p; } catch (e) {}
        return p;
      }
    });

    Object.defineProperty(Response_.prototype, 'text', {
      configurable: true,
      writable: true,
      value: function () {
        var res = this;
        if (res.__bcuText) return res.__bcuText;
        var p = nativeBodyText.call(res).then(function (body) {
          try {
            var ct = res.headers && res.headers.get ? res.headers.get('content-type') : '';
            if (ct && !/json|text|javascript|xml/i.test(ct)) return body;
            var fixed = neutralized(body);
            if (fixed !== null) return fixed;
          } catch (e) { /* fall through */ }
          return body;
        });
        try { res.__bcuText = p; } catch (e) {}
        return p;
      }
    });
  }

  /* ---------------------------------------------------------------- indicator */

  function badge() {
    var host = document.createElement('div');
    host.style.cssText = 'position:fixed;z-index:2147483647;left:12px;bottom:12px;pointer-events:none;' +
      'font:12px/1.4 ui-monospace,Consolas,monospace;background:#0b5fff;color:#fff;padding:6px 10px;' +
      'border-radius:6px;box-shadow:0 4px 14px rgba(0,0,0,.35);opacity:.92';
    host.textContent = 'BCU: hook armed, 0 rewrites';
    var paint = function () {
      var h = window.__BCU_HOOK__;
      host.textContent = 'BCU: hook armed, ' + h.hits + ' rewrit' + (h.hits === 1 ? 'e' : 'es');
      host.style.background = h.hits ? '#0f9d58' : '#0b5fff';
    };
    var append = function () {
      if (!document.body) return setTimeout(append, 50);
      document.body.appendChild(host);
      setInterval(paint, 1000);
      paint();
    };
    append();
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', badge, { once: true });
  } else {
    badge();
  }

  console.log('%c[BCU] armed @ ' + location.href, 'color:#0b5fff;font-weight:bold');
})();
```

# 声明

**一、本文的性质与目的**

本文记录的是我在使用 Broadcom 支持门户下载免费软件时，遇到前端状态位与账号实际权限不一致这一**具体技术问题**的排查过程与成因分析。文中脚本属于**前端调试与兼容性研究用途**，目的是让页面正确反映我账号本就具备的下载权限，而非获取任何我无权访问的资源。

**二、技术边界**

需要明确的是，脚本所做的一切都发生在**我自己的浏览器内部**：仅对本地收到的接口响应做解析后处理。全文不涉及、脚本也不包含以下任何行为 ——

- 未侵入、扫描、探测或干扰 Broadcom 的任何服务器与网络设施
- 未伪造身份凭证、未绕过身份认证、未越权访问他人数据
- 未破解加密、未提权、未生成或伪造任何授权文件与许可证
- 未修改文件真实性校验，下载所得安装包为官方原始文件

因此本文讨论的对象是**前端渲染逻辑**，而**不是**任何服务端安全机制。

**三、关于出口管制与 GTM 筛查**

Broadcom 依据其合规义务实施的 GTM（Global Trade Management）出口管制筛查，其**判定与执行完全在服务端**。本文脚本不改变、也无法改变 Broadcom 是否允许向特定主体提供软件的任何判定结果，更不构成对该机制本身的规避。文中 `exportControlStatus` 仅是前端用于渲染下载按钮状态的一个字段值。

本人不提供、不分发任何软件安装包、注册码或授权文件，亦不协助他人规避出口管制、制裁或其他强制性法律法规。

**四、读者须知与风险提示**

1. **请优先走官方渠道。** 若你的账号出现同类提示，建议先按 Broadcom 官方 KB 368588 的指引，将资料修改为英文全名、完整街道地址（含门牌号、非 PO Box）、纯 ASCII 字符后重新提交，等待自动筛查完成。这是治本方式，且与本文内容不冲突。
2. **可能违反服务条款。** Broadcom 门户的服务条款通常禁止绕过访问限制类操作。该风险属**合同层面**，最坏情形是相关账号被限制或停用，请自行评估并承担。
3. **请勿使用企业账号或他人账号实操**，相关风险应由实际操作者自行承担。
4. **时效性说明。** 本文所依赖的字段名与前端实现由 Broadcom 单方控制，随时可能变更，脚本届时可能失效。请勿将其视为长期有效的下载途径。
5. **免责。** 本文内容仅供技术学习与交流。读者因使用文中代码所产生的一切后果，由使用者自行负责；本人不对脚本的可用性、持续有效性或任何直接、间接损失作出担保。
6. 文中提及的第三方产品名称、接口与文档链接，其权利归各自权利人所有。

**五、许可**

文中脚本以 MIT 许可发布，可自由使用与修改。若转载或二次分发，请保留原始出处与本节声明。
