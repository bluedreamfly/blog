---
title: 升级webpack2及构建速度优化
date: 2017-05-15
categories: ["技术", "前端", 'webpack2']
tags: ["webpack"]
---
&emsp;&emsp;`webpack2`很早就已经放出了`2.0beta`版本，但是还不稳定，所以在生产上也没敢用上。虽然正式版也放出了一段时间，但也没空去做升级。
直到业务内部系统的业务一直迭代，项目膨胀得太快。发现构建速度慢得一逼，简直不能忍，所以决心要优化一下webpack的配置，顺便升级`webpack2`及`React15.5`带来的一系列变化。

&emsp;&emsp;现在我们的项目代码主要分为两部分，一部分是第三方库，一部分是业务代码。第三方库是通过`CommonsChunkPlugin`这个插件跟业务代码分离，但这实际上只是解决代码层面的问题，就是如果只有业务代码更新，我们不需要重新下载第三方库的代码。但是在项目构建过程当中，仍然是会一起构建的，所以这并不能达到提升构建速度的目的。

&emsp;&emsp;我们现在需要解决的是第三方库如果不升级，那么我们是不需要重复去构建的，基本上只需要构建一次，除非版本升级。而发现webapck也提供了这样的一个插件`DllPlugin`和`DllReferencePlugin`，`DllPlugin`这个插件主要来打包第三方共有库，然后生成一个`mainifest.json`的文件，所以我们需要有一个配置专门来打包和生成映射文件。

```javascript
//webpack.config.dll.js
module.exports = {
  entry: {
    vendor: [
      'react',
      'react-dom',
      'react-redux',
      'react-router',
      'react-router-redux',
      'redux',
      ...
    ]
  },
  output: {
    ...
  },
  plugins: [
    new webpack.DllPlugin({
    path: 'build/'+ (isDev ? 'dev.' : '') + '[name]-manifest.json',
    name: '[name]_lib'
  }),
  ]
}

```
&emsp;&emsp;这份配置会生成本地开发跟线上需要的版本，本地的第三方库是不发到`cdn`，我这边是会生成六个文件`dev.index.html`、`dev.vendor-mainifest.json`、`dev.vendor.[hash].js`、`index.html`、`vendor-mainifest.json`、`vendor.[hash].js`，带`dev`前缀的是本地开发需要用到的。这边为什么有两份`html`的原因是后面需要内联不同环境的`js`文件。其中mainifest文件是编译业务代码需要用到的一个库映射表。

所以我们还需要有个业务代码的配置文件。
```javascript

module.exports = {
  context: path.join(__dirname, './src'),
  
  output: {
    path: path.join(__dirname, './dist'),
    filename: '[name].js',
  },
  module: {
    rules: [
      ...
    ]
  },
  plugins: [
    new webpack.DllReferencePlugin({
      context: __dirname,
      manifest: require(path.resolve('./build/' + (isDev ? 'dev.' : '') + 'vendor-manifest.json'))
    })
  ]
}

```
通过这上面的优化基本上从原有的`27`秒减到`20`秒，看到这个点还是有点小激动的，不过看看时间还是这么长，感觉还是有优化的空间的，所以决定再找找方案。

其实在整个编译过程当中，时间基本是浪费在解析和压缩上面了，所以我们可以从这两个方面入手。我们先从解析这部分入手，其实解析的时候都是串行的，所以这样就会拖慢速度，那能不能通过并行解析道方式来提高速度呢？还真不出自己的猜想，确实有人写了这么一个插件`happypack`，对于一个文件较多的项目来说提升还是很明显的。我们可以这样用

```javascript
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: [
          'react-hot-loader/webpack',
          'happypack/loader'
        ]
      }
    ]
  },
  
  plugins: [
    new HappyPack({
      cache: true,
      loaders: [{
        loader: 'babel-loader',
      }],
      threads: 2
    })
  ]
```

在这个配置当中，我们只有`js`及`jsx`文件用了`happypack`，其实它可以用多个`happypack`实例来处理不同的文件，这个效果还是很明显的，虽然现在稍大一点的项目开发环境还是需要二十几秒，但相比当初好几分钟已经是质的飞跃了。所以，有时候没遇到痛点的时候我们往往得过且过。经过这一轮优化整体的构建速度快了一个数量级，对于大项目效果更明显。当然我们刚才讲过，在压缩过程，还是可以进行优化的，同样也是可以并行压缩的。











