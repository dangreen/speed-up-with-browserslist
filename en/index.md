# Speed up with Browserslist

Today we have large number of different browsers and even more versions of each of them. Not so long ago features were added to browsers infrequently, now you can see them with almost every new release. As a result, different versions of the browser have different features support, and this without considering different level of support among different vendors.

Developers want to use new features, as this often simplifies their life. Using modern development tools, you can use features before they appear in browsers by transpiling and using polyfills. Also, these tools guarantee that website will work any in browser, regardless it has support for a particular feature. Tools examples: [Autoprefixer](https://github.com/postcss/autoprefixer) and [postcss-preset-env](https://preset-env.cssdb.org) for CSS, [Babel](https://babeljs.io) for JavaScript. But you need to understand that using this tools can increases size of the code.

As a result, we have a website that works in any browser, but it loads more slowly than it could. Let me remind you that speed of website loading and working directly affects number and depth of views. What can be done with this? In fact, we don’t need to transpile and polyfill absolutely all features — it’s enough to do this only with those that are not supported by current browsers (or relevant to the audience of your website). For example, promises are not supported only by very old versions of browsers.

## Browserslist

[Browserslist](https://github.com/browserslist/browserslist) — a tool to describe target browsers of web application using simple queries:

```
last 2 years
> 1%
not dead
```

This is example of `.browserslistrc` file means you need live browsers over past two years, with more than 1% of users. You can see in which specific browsers this will be resolved on [browserl.ist](https://browserl.ist/), and you can learn more about queries syntax [on project page](https://github.com/browserslist/browserslist).

Already mentioned [Autoprefixer](https://github.com/postcss/autoprefixer), [postcss-preset-env](https://preset-env.cssdb.org) and [babel-preset-env](https://babeljs.io/docs/en/babel-preset-env) under the hood use Browserslist, and if your project has a Browserslist config, project code will be compiled for these browsers.

At this stage, we can come to following conclusion: than newer browsers we targeting at, than less the final code of website will weigh. At the same time, we should not forget that in real world not all users have newest browser, and website should be accessible to all users, or most of them. What can be done under these conditions?

## Browser targeting variants

### 1. Limited targeting

By default, if there is no config in the project, then Browserslist will use `defaults` browsers. This query is alias for `> 0.5%, last 2 versions, Firefox ESR, not dead`. In general, you can stop on this query, and over time, browsers matching this query will begin support most of current features.

But you can targeting at limited number of browsers: exclude legacy and unpopular ones, consider more or less relevant versions of browsers. Sounds simple, but actually it is not. You need to balance Browserslist config so that, to cover most of audience.

### 2. Audience analysis

If your website implies use only in certain regions, then you can try using query like `> 5% in US`, which will return browsers that are suitable by usage statistics in specified country.

Browserslist family is full of various additional tools, one of them is [Browserslist-GA](https://github.com/browserslist/browserslist-ga) (there is also [browserslist-adobe-analytics](https://github.com/xeroxinteractive/browserslist-adobe-analytics)), which allows you to export data about browsers used by website visitors from analytics service. After that, becomes possible to use this data in Browserslist config and make queries on them:

```
> 0.5% in my stats
```

For example, if update this data on every deploy, then website will always built for current browsers used by your audience.

### 3. Differential resource loading

In March 2019 [Matthias Binens](https://twitter.com/mathias) from Google [proposed](https://github.com/whatwg/html/issues/4432) to add differential script loading (further DSL) to browsers:

```html
<script type="module"
        srcset="2018.mjs 2018, 2019.mjs 2019"
        src="2017.mjs"></script>
<script nomodule src="legacy.js"></script>
```

Until now, his proposal stay only a proposal, and it is not known whether this will be implemented in browsers. But concept is understandable, and Browserslist family has tools with which you can implement something similar, one of them is [browserslist-useragent](https://github.com/browserslist/browserslist-useragent). This tool allows you to check browser’s User-Agent on satisfying your config.

#### Browserslist-useragent

There are already several articles on this topic, here is an example of one of them — [«Smart Bundling: How To Serve Legacy Code Only To Legacy Browsers»](https://www.smashingmagazine.com/2018/10/smart-bundling-legacy-code-browsers/). Briefly go over implementation. First you need to configure your build process to output, for example, two versions of bundles for new and legacy browsers. Here Browserslist will help you with its [ability to declare several environments in configuration file](https://github.com/browserslist/browserslist#configuring-for-different-environments):

```
[modern]
last 2 versions
last 1 year
not safari 12.1

[legacy]
defaults
```

Next, you need to configure server to send right bundle to user’s browser:

```js
/* … */
import { matchesUA } from 'browserslist-useragent'
/* … */
app.get('/', (request, response) => {
    const userAgen = request.get('User-Agent')
    const isModernBrowser = matchesUA(userAgent, {
        env: 'modern',
        allowHigherVersions: true
    })
    const page = isModernBrowser
        ? renderModernPage(request)
        : renderLegacyPage(request)

    response.send(page)
})
```

Thus, website will send lightweight bundle to users with new browsers, as result faster loading speed of pages, while saving accessibility for other users. But, as you can see, this method requires own server with special logic.

#### Module/nomodule

With browsers support of ES-modules, was found way to implement DSL on client side:

```html
<script type="module" src="index.modern.js"></script>
<script nomodule src="index.legacy.js"></script>
```

This pattern called module/nomodule, and its work is that legacy browsers without ES-modules support will not handle scripts with type `module`, since this type is unknown to them. And browsers that support ES-modules will load scripts with type `module` and ignore scripts with `nomodule` attribute. Browsers with ES-modules support can described by following config:

```
[esm]
edge >= 16
firefox >= 60
chrome >= 61
safari >= 11
opera >= 48
```

Main advantage of module/nomodule pattern is that you doesn’t need own server — everything works completely on client side. Differential stylesheet loading cannot be done by this way, but you can implement resource loading using JavaScript:

```js
if ('noModule' in document.createElement('script')) {
    // Modern browsers
} else {
    // Legacy browsers
}
```

One of disadvantages: this pattern has some [cross-browser problems](https://gist.github.com/jakub-g/5fc11af85a061ca29cc84892f1059fec). Also, browsers that support ES-modules already have new features with different level of support, for example, [optional chaining operator](https://caniuse.com/#feat=mdn-javascript_operators_optional_chaining). With addition of new features, this DSL variation will lose its relevance.

You can read more about module/nomodule pattern in the article [«Modern Script Loading»](https://jasonformat.com/modern-script-loading/). If you are interested in this DSL variant and would like to try it in your project, then you can use Webpack plugin: [webpack-module-nomodule-plugin](https://www.npmjs.com/package/webpack-module-nomodule-plugin).

#### Browserslist-useragent-regexp

More recently, another tool was created for Browserslist: [browserslist-useragent-regexp](https://github.com/browserslist/browserslist-useragent-regexp). This tool allows you to get a regular expression from config to check 
browser’s User-Agent. Regular expressions work in any JavaScript runtime, which makes it possible to check browser’s User-Agent not only on server side, but also on client side. Thus, you can implement a working DSL in browser:

```js
// last 2 firefox versions
var modernBrowsers = /Firefox\/(73|74)\.0\.\d+/
var script = document.createElement('script')

script.src = modernBrowsers.test(navigator.userAgent)
    ? 'index.modern.js'
    : 'index.legacy.js'

document.all[1].appendChild(script)
```

Another fact that generated regular expressions are faster than matchesUA function from browserslist-useragent, so it makes sense to use browserslist-useragent-regexp on server side too:

```js
> matchesUA('Mozilla/5.0 (Windows NT 10.0; rv:54.0) Gecko/20100101 Firefox/54.0', { browsers: ['Firefox > 53']})
first time: 21.604ms
> matchesUA('Mozilla/5.0 (Windows NT 10.0; rv:54.0) Gecko/20100101 Firefox/54.0', { browsers: ['Firefox > 53']})
warm: 1.742ms

> /Firefox\/(5[4-9]|6[0-6])\.0\.\d+/.test('Mozilla/5.0 (Windows NT 10.0; rv:54.0) Gecko/20100101 Firefox/54.0')
first time: 0.328ms
> /Firefox\/(5[4-9]|6[0-6])\.0\.\d+/.test('Mozilla/5.0 (Windows NT 10.0; rv:54.0) Gecko/20100101 Firefox/54.0')
warm: 0.011ms
```

All this looks very cool, but there should be easy way to integrate it into project’s building process... And there is such a way!

#### Browserslist Differential Script Loading

[Bdsl-webpack-plugin](https://github.com/TrigenSoftware/bdsl-webpack-plugin) is a Webpack plugin paired with [html-webpack-plugin](https://github.com/jantimon/html-webpack-plugin) and using [browserslist-useragent-regexp](https://github.com/browserslist/browserslist-useragent-regexp), which helps automate adding DSL to the bundle. Here is example Webpack config for using this plugin:

```js
const {
    BdslWebpackPlugin,
    getBrowserslistQueries,
    getBrowserslistEnvList
} = require('bdsl-webpack-plugin')

function createWebpackConfig(env) {
    return {
        name: env,
        /* … */
        module: {
            rules: [{
                test: /\.js$/,
                exclude: /node_modules/,
                loader: 'babel-loader',
                options: {
                    cacheDirectory: true,
                    presets: [
                        ['@babel/preset-env', {
                            /* … */
                            targets: getBrowserslistQueries({ env })
                        }]
                    ],
                    plugins: [/* … */]
                }
            }]
        },
        plugins: [
            new HtmlWebpackPlugin(/* … */),
            new BdslWebpackPlugin({ env })
        ]
    };
}

module.exports = getBrowserslistEnvList().map(createWebpackConfig)
```

This example exports [several configs](https://webpack.js.org/configuration/configuration-types/#exporting-multiple-configurations) to output bundles for each environment from Browserslist config. At output, we get HTML file with built-in DSL script:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Example</title>
        <script>function dsl(a,s,c,l,i){c=dsld.createElement('script');c.async=a[0];c.src=s;l=a.length;for(i=1;i<l;i++)c.setAttribute(a[i][0],a[i][1]);dslf.appendChild(c)}var dsld=document,dslf=dsld.createDocumentFragment(),dslu=navigator.userAgent,dsla=[[]];if(/Firefox\/(73|74)\.0\.\d+/.test(dslu))dsl(dsla[0],"/index.modern.js")
else dsl(dsla[0],"/index.legacy.js");dsld.all[1].appendChild(dslf)</script>
    </head>
    <body></body>
</html>
```

In addition to scripts loading, there is support for [styles loading](https://github.com/TrigenSoftware/bdsl-webpack-plugin/tree/master/examples/postcss-preset-env). It is also possible to use this plugin on [server side](https://github.com/TrigenSoftware/bdsl-webpack-plugin/tree/master/examples/SsrBdslWebpackPlugin).

But, unfortunately, there are some nuances that you should know before start using bdsl-webpack-plugin: since scripts and styles loading is initialized from JavaScript, they are loading asynchronously without render blocking, and etc. For example, in case of scripts — this means inability to use `defer` attribute, and for styles — need to hide page content until styles are fully loaded. About how to get around these nuances, and about other features of this plugin, see [documentation](https://github.com/TrigenSoftware/bdsl-webpack-plugin/blob/master/README.md) an [usage examples](https://github.com/TrigenSoftware/bdsl-webpack-plugin#examples).

## Dependencies transpilation

From already read part of the article, we learned several ways of using Browserslist to reduce size of _own_ code of website, but other part of the final bundle is dependencies. In web applications, size of dependencies in the final bundle can take up a significant part.

By default, build process should avoid transpilation of dependencies, since then the build will take a lot of time, and dependencies, if their sources use unsupported syntax, are usually distributed already transpiled. In practice, there are three types of packages:

1. With transpiled code;
2. With transpiled code and sources;
3. With code with current syntax only for new browsers.

With the first type, obviously, nothing can be done. The second — you need to configure bundler to work only with sources from package. The third type — for code to work, even in not very relevant browsers, you still need to transpile it.

Since there is no common way to make packages with several versions of bundle, I will describe how I make such packages: regular transpiled version has `.js` extension, and main file is written in `main` field of `package.json` file, and version of bundle without transpilation has `.babel.js` extension, and main file is written in `raw` field. Here is real example — [Canvg](https://unpkg.com/browse/canvg/) package. But you can do it in another way, for example, how it done [in Preact package](https://unpkg.com/browse/preact/) — sources are located in separate folder, and in `package.json` there is `source` field.

For Webpack can work with such packages, you need to modify `resolve` config section:

```js
{
    /* … */
    resolve: {
        mainFields: [
            'raw',
            'source',
            'browser',
            'module',
            'main'
        ],
        extensions: [
            '.babel.js',
            '.js',
            '.jsx',
            '.json'
        ]
    }
    /* … */
}
```

In this way, we tell Webpack how it should lookup files in packages that it should use for build. Then we just need to configure [babel-loader](https://github.com/babel/babel-loader):

```js
{
    /* … */
    test: /\.js$/,
    exclude: _ => /node_modules/.test(_) && !/(node_modules\/some-modern-package)|(\.babel\.js$)/.test(_),
    loader: 'babel-loader'
    /* … */
}
```

The logic is simple: we ask to ignore everything from `node_modules`, except specific packages and files with specific extension.

## Results

I measured size of [DevFest Siberia 2019](https://github.com/TrigenSoftware/DevFest-Siberia) website’s bundle and loading speed before and after applying differential loading together with dependencies transpilation:

<table>
    <thead>
        <tr>
            <th></th>
            <th>Regular network</th>
            <th>Regular 4G</th>
            <th>Good 3G</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th colspan="4">Without DSL</th>
        </tr>
        <tr>
            <td>Average loading time</td>
            <td>1,511 ms</td>
            <td>4,240 ms</td>
            <td>8,696 ms</td>
        </tr>
        <tr>
            <td>Fastest loading time</td>
            <td>1,266 ms</td>
            <td>3,366 ms</td>
            <td>8,349 ms</td>
        </tr>
        <tr>
            <td>Encoded size</td>
            <td colspan="3">292 kB</td>
        </tr>
        <tr>
            <td>Decoded size</td>
            <td colspan="3">1.08 MB</td>
        </tr>
        <tr>
            <th colspan="4">bdsl-webpack-plugin, 3 environments (modern, actual, legacy)</th>
        </tr>
        <tr>
            <td>Average loading time</td>
            <td>1,594 ms</td>
            <td>3,409 ms</td>
            <td>8,561 ms</td>
        </tr>
        <tr>
            <td>Fastest loading time</td>
            <td>1,143 ms</td>
            <td>3,142 ms</td>
            <td>6,673 ms</td>
        </tr>
        <tr>
            <td>Encoded size</td>
            <td colspan="3">218 kB</td>
        </tr>
        <tr>
            <td>Decoded size</td>
            <td colspan="3">806 kB</td>
        </tr>
    </tbody>
</table>
Result is increased loading speed and size of the bundle decreased by ≈20%, [read more detailed report](https://gist.github.com/dangreen/5427c5f2158c357bf0b15d38270508ac). You can also make measurements by yourself — necessary script you can find [in bdsl-webpack-plugin repository](https://github.com/TrigenSoftware/bdsl-webpack-plugin#metrics).

### Sources

- [Smart Bundling: How To Serve Legacy Code Only To Legacy Browsers](https://www.smashingmagazine.com/2018/10/smart-bundling-legacy-code-browsers/), Shubham Kanodia
- [Modern Script Loading](https://jasonformat.com/modern-script-loading/), Jason Miller

### Author
- [Dan Onoshko](https://twitter.com/dangreen58)

### Editor
- [Vadim Makeev](https://twitter.com/pepelsbey)
- Irina Pitaeva

### Translation
- [Dan Onoshko](https://twitter.com/dangreen58)
