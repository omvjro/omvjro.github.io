---
title: "Mediawiki 语言变体补遗"
description: 利用 Gadget 和 pybot
date: 2023-12-16
categories:
  - MediaWiki
tags:
  - Python
  - JavaScript
draft: true
---

如[先前](/p/mediawiki-语言变体权宜之计/)所述，利用 JavaScript 实现的变体转换存在需要额外创建繁体标题重定向的问题。和更改站点内容语言一样，对于已有一定规模的 wiki 来说，这是很麻烦的一件事。

## 手动创建重定向

利用此脚本可以为每个页面添加快速创建繁体重定向的按钮。

```js
$(function () {
const ifExist = title => {
  return new Promise(resolve => {
    new mw.Api().get({
      action: 'parse',
      page: title,
    }).done(() => {
      resolve(1);
    }).fail(() => {
      resolve(0);
    });
  });
}

const getEntitle = title => {
  return new Promise(resolve => {
    new mw.Api().get({
      action: 'parse',
      page: title,
      redirects: true,
      prop: 'wikitext',
      format: 'json',
    }).done((data) => {
      let wikitext = data.parse.wikitext['*'];
      let entitlep, entitle;
      if (wikitext.indexOf('{{en|') > -1) {
        entitlep = wikitext.slice(wikitext.indexOf('{{en|'));
      } else if (wikitext.indexOf('{{En|') > -1) {
        entitlep = wikitext.slice(wikitext.indexOf('{{En|'));
      } else { resolve(0); }
      entitle = entitlep.slice(5, entitlep.indexOf('}}'));
      resolve(entitle);
    }).fail(() => {
      resolve(0);
    });
  });
}

$('#site-tools ul').append('<li class="mw-list-item" id="t-tradredirect"><a href="javascript:;">创建繁体重定向</a></li>');

document.getElementById('t-tradredirect').addEventListener('click', async () => {
  const msg = m => document.getElementById('t-tradredirect').innerHTML = m;
  msg('请稍候');
  let title = mw.config.get('wgPageName');
  const converter = OpenCC.Converter({ from: 'cn', to: 'tw' });
  let tradtitle = converter(title);
  if (tradtitle == title) {
    msg('无需创建繁体重定向');
    return;
  }
  if (await ifExist(tradtitle)) {
    msg('已有繁体重定向');
    return;
  }
  console.log(`${tradtitle} => ${title}`);
  new mw.Api().postWithToken('csrf', {
    action: 'edit',
    text: `#重定向 [[${title}]]`,
    title: tradtitle,
    minor: false,
    nocreate: false,
    summary: `创建繁体重定向`,
    errorformat: 'plaintext'
  }).done(() => {
    msg('创建成功');
  }).fail((a, b, errorThrown) => {
    msg('出错了');
    console.log(errorThrown);
  });
});
});
```

## Gadget

可以利用 Gadget 将脚本设置为可供用户启用。

- [Extension:Gadgets](https://www.mediawiki.org/wiki/Extension:Gadgets)

## Pybot

使用 Pywikibot 可以更妥当地批量处理先前的页面。

- [帮助:使用Python编辑](https://mzh.moegirl.org.cn/Help:%E4%BD%BF%E7%94%A8Python%E7%BC%96%E8%BE%91)
- [Manual:Pywikibot](https://www.mediawiki.org/wiki/Manual:Pywikibot)
