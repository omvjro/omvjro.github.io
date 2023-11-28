---
title: "MediaWiki 语言变体另辟蹊径"
description: 在维基农场上为初始语言并非 zh 的站点添加语言变体功能
date: 2023-11-28
categories:
  - MediaWiki
tags:
  - JavaScript
---
在 [DOL 中文 wiki](https://degreesoflewditycn.miraheze.org/) 中，或由于初始设置失误，语言设置为 `zh-CN` 而非 `zh`。根据 [MediaWiki 文档](https://www.mediawiki.org/wiki/Manual:$wgDisableLangConversion/zh)，只有语言设置为 `zh` 才能启用自带的变体转换功能（以 `$wgDisableLangConversion` 设置）。

## 用户界面语言和站点内容语言

首先需由[文档](https://www.mediawiki.org/wiki/Manual:Language)明确，MediaWiki 的语言分为三种：

- 站点内容语言（Site content language）
- 用户界面语言（User interface language，API 调用为 `mw.user.options.values.language`）
- 页面内容语言（Page content language）

其中页面内容语言极少需要配置，绝大部分情况下，都默认与 `站点内容语言` 一致，在此忽略。

`站点内容语言` 决定了 `html` 的 `lang` 属性，在默认情况下，也决定了 wiki 具体内容的语言，实际使用上，主要有标题（`#firstHeading`）和内容（`#mw-content-text`）两部分。

`用户界面语言` 决定了基础界面的 `lang` 属性，这些基础界面也就是“用户界面消息”（User interface message）。用户界面消息大部分由 MediaWiki 自动生成，例如命名空间的“User”（`en`）和“用户”（`zh`），能够随 `用户界面语言` 改变自己改变语言；但有一部分用户界面消息来自 MediaWiki 命名空间，最常用的如 `MediaWiki:Sidebar` 为侧边栏（`#site-navigation`），其 `lang` 属性虽能随 `用户界面语言` 改变而改变，具体内容却不会变化。

## 清除用户界面信息缓存

> The site content language (`ContentLanguage` service in `MediaWiki\MediaWikiServices::getContentLanguage`, based on `$wgLanguageCode`), which **should generally stay the same as long as the wiki exists**.

作为往往在每个页面都有使用的内容，用户界面消息有着强大的缓存。根据 [MediaWiki 文档](https://www.mediawiki.org/wiki/Manual:$wgLanguageCode)，更改 wiki 语言（`$wgLanguageCode`）后，需清除用户界面消息缓存，否则会无法显示新语言界面消息，或新旧语言界面消息混杂。其中提供了两种方法。

在 MediaWiki 1.18 及以上，运行

```shell
echo 'MediaWiki\MediaWikiServices::getInstance()->getMessageCache()->clear()' | php maintenance/eval.php
```

在 MediaWiki 1.18 以下，则手动运行 `maintenance/rebuildmessages.php`。

然而，在 Miraheze 这样的维基农场中，往往无法如此清除缓存。

## 设置 `$wgForceUIMsgAsContentMsg`

经指点并查[文档](https://www.mediawiki.org/wiki/Manual:$wgForceUIMsgAsContentMsg)，亦可以设置 `$wgForceUIMsgAsContentMsg`，将用户界面消息设置为内容消息，即能够自己转换。

显然，在维基农场中，也不可改动这个设置。

## 编写自定义变体脚本

试模拟原变体功能，在内容页面右上排按钮（`.mw-list-item`）中添加变体按钮，加入全站加载的 `MediaWiki:Common.js` 中。

### opencc-js

[OpenCC](https://github.com/BYVoid/OpenCC) 是非常常用的一个简繁转换工具，凭借巨大的词库，可以实现各地不同繁体的转换，故欲以此编写转换脚本。原工具为 npm 包，难以直接使用，但有衍生 JavaScript 版本 [opencc-js](https://github.com/nk2028/opencc-js)，此处则使用此版本。

摘抄用法如下：

```js
// Set Chinese convert from Traditional (Hong Kong) to Simplified (Mainland China)
const converter = OpenCC.Converter({ from: 'hk', to: 'cn' });
// Set the conversion starting point to the root node, i.e. convert the whole page
const rootNode = document.documentElement;
// Convert all elements with attributes lang='zh-HK'. Change attribute value to lang='zh-CN'
const HTMLConvertHandler = OpenCC.HTMLConverter(converter, rootNode, 'zh-HK', 'zh-CN');
HTMLConvertHandler.convert(); // Convert  -> 汉语
HTMLConvertHandler.restore(); // Restore  -> 漢語
```

由此按照 `lang` 属性转换的特性，可以为需转换内容设置不存在的 `lang` 属性，如 `zh-to-convert`，实现不影响其他部分的转换；对于易误转换的内容，如 `textarea`，也可以设置如 `zh-not-convert` 的 `lang` 属性，随后用 `restore` 方法转回原文。

特别是在 MediaWiki 中，需要转换的元素主要为标题（`#firstHeading`）、侧边栏（`#site-navigation`）、内容（`#mw-content-text`），这些部分的特点是具有 MediaWiki 根据 `站点内容语言` 或 `用户界面语言` 设定设置的 `lang` 属性；然而，`html` 同样拥有此属性，因此不能直接根据原属性转换。

opencc-js 亦支持自定义词典，在 wiki 实际使用中应很有用处，但暂不研究。

### 外部脚本加载问题

MediaWiki 中，传统的引入外部脚本的方式为使用 `mw.loader.load`。

```js
mw.loader.load( 'https://cdn.jsdelivr.net/npm/opencc-js@1.0.5/dist/umd/full.js' );
```

然而，当网络情况不佳或脚本稍大（如 opencc-js）时，会出现未加载完成的情况。

幸而，在 MediaWiki 1.33 中，新增了一种引入外部脚本的方式，为 `mw.loader.getScript`，其会传回一个 Promise 对象。无需知道 Promise 对象的含义，只需参考示例代码，就能实现外部脚本加载完后执行函数。

```js
mw.loader.getScript( 'https://cdn.jsdelivr.net/npm/opencc-js@1.0.5/dist/umd/full.js' )
.then(function(){
    // 之后执行的内容
});
```

### 只支持 ES5 的屑 MediaWiki

调试时可能出现报错，说明只能使用 ES5 语法。ES6 2015 年即推出，目前已经普遍支持，很容易不小心写出来。例如模板字符串、函数以 `=>` 表示等。

## 结果

以下为暂用的脚本，实现以 `localStorage` 存储用户偏好的变体转换，理论上可适用于所有 wiki，只需微调 CSS。

```js
mw.loader.getScript( 'https://cdn.jsdelivr.net/npm/opencc-js@1.0.5/dist/umd/full.js' )
.then(function(){
$(function(){
const cth = OpenCC.Converter({ from: 'cn', to: 'hk' });
const ttc = OpenCC.Converter({ from: 'tw', to: 'cn' });
const ctt = OpenCC.Converter({ from: 'cn', to: 'tw' });

$('#firstHeading').attr('lang', 'zh-to-convert');
$('#site-navigation ul').attr('lang', 'zh-to-convert');
$('#site-navigation h3').attr('lang', 'zh-to-convert');
$('#mw-content-text').attr('lang', 'zh-to-convert');
if($('textarea')) { $('textarea').attr('lang', 'zh-not-convert'); }

const rootNode = document.documentElement;
const HTMLConvertHandler = {
    "cn": OpenCC.HTMLConverter(ttc, rootNode, 'zh-to-convert', 'zh-CN'),
    "hk": OpenCC.HTMLConverter(cth, rootNode, 'zh-to-convert', 'zh-HK'),
    "tw": OpenCC.HTMLConverter(ctt, rootNode, 'zh-to-convert', 'zh-TW'),
}
if (localStorage.getItem('opencc')) {
    HTMLConvertHandler[localStorage.getItem('opencc')].convert();
}

$('<style>.mw-list-item select {color: #68d;padding: inherit;padding-right: 2em;background-color: transparent;cursor: pointer;border: none;}.mw-list-item select:hover {border: none;}</style>').appendTo('body');
$('<li class="mw-list-item"><select name="opencc" id="opencc"><option value="">不转换</option><option value="cn">大陆简体</option><option value="hk">港澳繁体</option><option value="tw">台湾繁体</option></select></li>').appendTo($('#p-views ul'));
if (localStorage.getItem('opencc')) {
    $('#opencc').val(localStorage.getItem('opencc'));
}
$('#opencc').change(function() {
    localStorage.setItem('opencc', event.target.value);
    location.reload();
});
});
});
```

效果如下：

![效果](https://p.sda1.dev/14/54b68f6a606d0f24cf6e52e6f530e875/QQ截图20231112182613.png)
