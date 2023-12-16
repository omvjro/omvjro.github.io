---
title: "Mediawiki 语言变体补遗"
description: 只是权宜之计
date: 2023-12-16
categories:
  - MediaWiki
tags:
  - JavaScript
---

如[先前](/p/mediawiki-语言变体权宜之计/#补遗)所述，利用 JavaScript 实现的变体转换存在需要额外创建繁体标题重定向的问题。和更改站点内容语言一样，对于已有一定规模的 wiki 来说，这是很麻烦的一件事。

## 手动创建脚本

利用此脚本可以为每个页面添加快速创建繁体重定向的按钮，所依赖的 API 有 jQuery、[openccjs](https://github.com/nk2028/opencc-js)，以及 MediaWiki 自身提供的 [Action API](https://www.mediawiki.org/wiki/API:Main_page)。

```js
$(function () {

// 仅在内容页面启用
if (!mw.config.get( 'wgIsArticle' ) || mw.config.get( 'wgPageContentModel' ) !== 'wikitext') { return }

// 检查重定向页面是否已存在
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

// 添加按钮至“Wiki工具”栏，可按需修改
$('#site-tools ul').append('<li class="mw-list-item" id="t-tradredirect"><a href="javascript:;">创建繁体重定向</a></li>');

document.getElementById('t-tradredirect').addEventListener('click', async () => {

  const msg = m => document.getElementById('t-tradredirect').innerHTML = m;
  msg('请稍候');

  let title = mw.config.get('wgPageName');
  // 使用了简体 -> 台繁，如有需要也可增添港繁（将 tw 修改为 hk ）
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
    summary: `创建繁体重定向`,
    errorformat: 'plaintext'
  }).done(() => {
    msg('创建成功');
  }).fail((a, b, errorThrown) => {
    msg('出错了');
    console.error(errorThrown);
  });

});

});
```

请注意 MediaWiki 仍然无法使用 ES6 语法，因此可将以上脚本通过油猴等方法加载以使用。

## Gadget

可以利用 Gadget 将脚本设置为可供用户启用，便于编辑者自助添加繁体重定向。此外，Gadget 似乎可以使用 ES6 语法。

- [Extension:Gadgets](https://www.mediawiki.org/wiki/Extension:Gadgets)

## 处理历史遗留问题

理论上，使用 Pywikibot 可以更妥当地批量处理先前的页面。然而，[Pywikibot 的文档](https://doc.wikimedia.org/pywikibot/stable/introduction.html)看上去就很麻烦，而且本人 Python 水平比 JavaScript 更低。鉴于 DOL 中文 wiki 页面尚不算多，直接照前文又画了一个 JavaScript 脚本了事。到 wiki 的 `特殊:所有页面`，在每一页打开控制台运行以下脚本即可。

建议使用机器人账户进行以下操作。

```js
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

// 获取需操作页面的标题
let link_list = [];
document.querySelectorAll('.mw-allpages-body li a').forEach( a => {

  // 排除重定向页面
  if (a.classList.contains('mw-redirect')) { return; }

  let title = a.innerText;
  const converter = OpenCC.Converter({ from: 'cn', to: 'tw' });

  // 排除简繁标题一致页面
  let tradtitle = converter(title);
  if (tradtitle == title) { return; }

  link_list.push(title);

});

link_list.forEach( async (title) => {

  const converter = OpenCC.Converter({ from: 'cn', to: 'tw' });
  let tradtitle = converter(title);
  if (await ifExist(tradtitle)) {
    console.log(`${tradtitle} => ${title}已有繁体重定向`);
    return;
  }

  new mw.Api().postWithToken('csrf', {
    action: 'edit',
    text: `#重定向 [[${title}]]`,
    title: tradtitle,
    bot: true, // 标记为机器人操作，避免刷屏
    summary: `创建繁体重定向`,
    errorformat: 'plaintext'
  }).done(() => {
    console.log(`${tradtitle} => ${title}创建成功`);
  }).fail((a, b, errorThrown) => {
    console.error(`${tradtitle} => ${title}创建出错`);
    console.error(errorThrown);
  });

});
```
