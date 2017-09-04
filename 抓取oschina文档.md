---
title: 抓取oschina文档
date: 2017-09-04 18:24
categories: ["技术", "爬虫"]
tags: ["爬虫"]
---

&emsp;&emsp;先问大家一个问题，大家平常都是用的哪些开发协作平台？我们公司用的是oschina。但我发现特别难用，特别关乎我们的是开发文档，如果后端同学没把文档结构搞好，我们后面找起来简直要命，关键是它不支持全文搜索，这一点真是深受其害。但是作为程序员这能忍吗？所以用了单身25年的手速撸出了一个简单的爬虫，把所有文档爬下来放在一起。虽然简单，但也基本能用，废话少说，开始撸。

&emsp;&emsp;说是开始撸，也总得有点想法不是？知己知彼，方能立于不败之地。首先就研究一下oschina上文档是如何生成的？实际上还是挺简单的。点进某一个项目的文档，就是一个侧边栏，侧边栏就是文档的目录，点击某一个节点右边就出现相关的内容（通过接口请求文档的html）,现在方向有了，那么就是一些技术问题。那具体实现思路是这样的，首先抓取某一文档项目的页面，通过分析页面的节点获取所有侧边栏对应文档的id, 然后根据id, 去获取文档的对应的html，再然后就是获取对应html的markdown，说挺简单的，做嘛就看个人了，我觉得也是挺简单的。想法思路都有了，那就挽起袖子开始撸。

&emsp;&emsp;那用什么来爬呢，对于前端莫过于node是最合适的了。所以先抓取项目文档页面

```javascript
const getHtml = (path) => {
  return new Promise((resolve, reject) => {
    const req = http.request({
      hostname: 'team.oschina.net',
      port: 80,
      path: path,
      method: 'GET',
      headers: {
        cookie: '这是你的cookie'
      }
    }, function(res) {

      let result = '';

      res.on('data', function(chuck) {
        result += chuck.toString();
      })
      res.on('end', function() {
        resolve(result);
      })
    })

    req.on('error', (e) => {
      console.error(`请求遇到问题: ${e.message}`);
      reject(e);
    });
    req.end();
  })
}

let html = await getHtml('/document/detail?p1=你的项目名&nv=doc');

```
接下来是解析对应的文档页面，获取不同节点的id

```javascript
const parse = (htmlStr) => {
  const dom = new JSDOM(htmlStr);
  
  let navs = dom.window.document.querySelectorAll('[data-ctype="2"]');
  let navlis = [];
  navs.forEach(nav => {
    navlis.push(nav.getAttribute('data-cid'));
  })
  return navlis; 
}
```

最后是将获取到的id去请求对应的文档页面

```javascript

const generateDOC = async () => {
  //获取项目对应的文档页面
  let result = await getHtml('/document/detail?p1=你的项目&nv=doc&dpid=24071&drd=216699');
  //解析节点
  let navs = parse(result);
  
  let docContent = [];
  let promises = [];
  //重新生成文档时清空doc.md文件
  fs.writeFileSync('./data/doc.md', '');
  promises = navs.map(async (nav, i) => {
    let apiTemp = await getHtml(`/action/document/preview_current_text?textId=${nav}&p1=你的项目&drd=832489`);
    let dom = new JSDOM(apiTemp);
    let mdContent = dom.window.document.querySelector('textarea').innerHTML;
    let hljsDom = dom.window.document.querySelectorAll('.hljs');
    let content = '';
    //对不规范的文档格式特殊处理
    if (hljsDom.length == 0) {
      mdContent.replace(/&nbsp;/gmi, ' ')
        .replace(/&gt;/gmi, '>')
        .replace(/^(#+)([^\s#])/mg, '$1 $2')
        .replace(/---/gmi, '')
      content = markdown(`### <a name="${nav}"></a>\n${mdContent}`)
    } else {
      let html = dom.window.document.querySelectorAll('#d_preview_area')[0].innerHTML;
      html = html.replace(/<textarea [^>]+>(.*)<\/textarea>/gm, (match, p1) => {
        return `<h5>${p1.replace(/#+/, '')}</h5>`
      });
      console.log(html);
      content = `<a name="${nav}"></a>\n${html}`;
      // console.log(content);;
    }
    
    docContent[i] = content;
  })

  await Promise.all(promises);
  let content = docContent.join('\n\n\n')
  fs.writeFileSync('./data/doc.md', content);
  return result;
}

```

其实这样已经可以生成整个文档的接口，但是需要将markdown转成html，并且相应高亮

```javascript

const markdown = (content) => {
  const renderer = new marked.Renderer();
  renderer.code = (code, language) => {
    
    const validLang = !!(language && hljs.getLanguage(language));
    
    const highlighted = validLang ? hljs.highlight(language, code).value : code;
    
    return `<pre><code class="hljs ${language}">${highlighted}</code></pre>`;
  };
  marked.setOptions({
    highlight: function (code) {
      // console.log(code);
      return hljs.highlightAuto(code).value;
    },
    renderer
  })
  return marked(content);
}

```

后面为了把侧边栏也生成出来

```javascript
  const generateNav = async () => {
  let detailHtml = await getHtml('/document/detail?p1=qiji&nv=doc&dpid=24071&drd=216699');
  let dom = new JSDOM(detailHtml);
  let nav = [];
  let document = dom.window.document;
  const iterate = (uls, parent) => {
    uls.forEach(ul => {
      let lis = document.querySelectorAll(`#${ul.id} > li`);

      lis.forEach(li => {
        let type = li.getAttribute('data-ctype');
        let cid = li.getAttribute('data-cid');
        let obj = {
          id: cid
        }
        if (type == 1) {
          let name = document.querySelector(`#${li.id} .d_tree_node`).getAttribute('title');
          obj.name = name;
          obj.children = [];
          parent.push(obj);
          iterate(document.querySelectorAll(`#${li.id} > ul`), obj.children);

        } else {
          let name = document.querySelector(`#${li.id} .d_tree_node_text`).getAttribute('title');
          obj.name = name;
          parent.push(obj);
        }
      })
    })
  }

  iterate(document.querySelectorAll('.d_tree > ul'), nav);
  fs.writeFileSync('./data/nav.json', JSON.stringify(nav));
  return nav;
}

```

这里面主要用到了三个三方库`jsdom`、`marked`、`highlight`这个三方库。其实现在谷歌浏览器出来一个`headless`模式的浏览器，可以通过这个去
抓取页面，并配合它对应nodejs库[puppeteer](https://github.com/GoogleChrome/puppeteer)，那时候就可以不用jsdom这个库，而且`puppeteer`这个库可以做的东西很多，后面有机会也写个关于这方面的使用。

