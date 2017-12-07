---
title: webpack学习笔记
tags: [webpack]
date: 2017-12-05 17:35:45
categories: 前端
---
### entry&output  
我们以一个最简单的配置为例：
```
    entry: {
        index:'./src/index.js',
    },
    output:{
        path: __dirname + '/dist',
        filename: '[name]__[chunkHash:8].js',
        chunkFilename: "[name]__[chunkHash:5]_chunk.js",
        publicPath: '/',
    }
```
首先看entry部分，该参数由若干键值对键值对构成，其中value部分可以传递数组，如：  
```
vendors:['react',react-dom]
```
大多数简单情况下，可以对entry配置两组键值对，其中一组为入口函数，另一组为所依赖的库文件，以便之后用webpack的插件 `CommonsChunkPlugin` 优化。  
  
output结构与entry相似：
-   path：这里的路径为webpack打包输出后的目录。其中 `__dirname` 属于关键字，可以用 `path` 这个包来替代。
-   filename：打包输出文件。 `[name]` 继承entry中key值，如 `index` , `vendors` 等作为输出文件名。  
    `[chunkhash]` 为该文件所含js模块的hash编码，有别于 `[hash]` 。hash为每次打包后生成的hash码，每当运行webpack时，所有文件重新编译，此hash重新生成，造成打包缓慢。  
    chunkhash为该JS模块的hash码，当编码过程中所修改的部分不涉及其他JS块时，其所对应的chunkhash不会改变，不会重复打包，故用此为好。
-   chunkFilename：模块输出文件名。应用在js文件存在异步加载模块中，以使    用最多的 `require.ensure` 为例：
    ```
        require.ensure([], (require) => {
            const { tip } = require('../common/messageTip');
            tip('require.ensure has been used');
        }, 'messageChunk');
    ```
    `require.ensure` 为异步加载JS代码，只有当执行时才进行获取指定JS文件。我们没有必要在首页加载时进行请求。webpack识别后抽取该代码块，会根据 `chunkFilename` 生成相应文件以便之后调用。
-   publicPath：文件引用路径。与之后 `HtmlwebpackPlugin` 紧密相关，当插件把打包后生成的文件路径插入html中，采取的是此路径。同时，当文件资源分开部署时，你也可以传递url对其引用。

### module
module中配置各种loader，对源代码进行编译打包。
```
    module: {
        rules: [{
            test: /\.jsx?$/,
            exclude: /node_modules/,
            loader: 'babel-loader',
        }, {
            test: /\.css$/,
            use: ['style-loader', 'css-loader', 'postcss-loader'],
        }, {
            test: /\.less$/,
            use: ['style-loader', 'css-loader', 'postcss-loader', 'less-loader'],
        }, {
            test: /\.(png|jpe?g|gif)(\?.+)?$/,
            loader: 'url-loader?name=image/[name].[hash:12].[ext]&limit=10000',
        }, {
            test: /\.(ttf|eot|svg|woff|woff2)(\?.+)?$/,
            loader: 'file-loader?name=font/[name].[hash:12].[ext]',
        }],
    }
```
对每个loader的作用做一个简单的阐述：
-   `babel-loader`  
    利用babel对es6（及以上）代码进行编译，生成浏览器可运行的js代码。所有规则在根目录下的 `.babelrc` 文件中。如下所示：
    ```
        {
            "presets": [
                [
                    "env",
                    {
                        "targets": {
                            "browsers": [
                                "last 2 versions",
                                "ie >= 9"
                            ]
                        },
                        "modules": false,
                        "useBuiltIns": true
                    }
                ],
                "stage-2",
                "react"
            ],
            "plugins": [
                "react-hot-loader/babel"
            ] //Enables React code to work with HMR.
        }
    ```
    我们使用了 `babel-preset-env/react/stage-2` 作为额外规则。  
    [babel-preset-env](https://github.com/babel/babel-preset-env) 作为新的preset，包括但不仅限于es2015的作用。
    >   A Babel preset that can automatically determine the Babel plugins and polyfills you need based on your supported environments.
-   `style-loader&css-loader&postcss-loader`  
    其中，`style-loader`和`css-loader`主要完成的功能是读取在js引用的css模块，之后插入输出页面中作为link。  
    `postcss-loader`作为前置，主要完成对所书写的css样式的再处理，添加各种浏览器兼容配置等，配置文件为 `postcss.config.js`：
    ```
        module.exports = {
            plugins: [
                require('autoprefixer')({
                    'browserslist': ['last 2 versions', 'ie>=9'],
                }),
            ],
        };
    ```
    `autoprefixer`为自带依赖，当然也可以执行npm install并记录在dev中。  
    `less-loader`作为css预处理，不再细说。  
-   `url-loader&file-loader`  
    两者功能作用大体相同。在这里，`url-loader`把在limit大小(KB)限制之内的图片转成base64编码插入相应位置，超过此大小的图片重新命名，打包到output指定目录下。同时插入hash码进行缓存控制。（有别于传统img标签内的src链接，`url-loader` 需要识别 `import` 引入的文件）。  
    `file-loader` 与之类似，在此用作字体等文件的加载。
### plugin
配置打包所需各种插件，自动化或优化某些过程。目前所用到配置如下：
```
    plugins: [
        new webpack.optimize.ModuleConcatenationPlugin(),
        new webpack.optimize.CommonsChunkPlugin({
            name: 'vendors',
            filename: 'common.js',
            minChunks: Infinity,
        }),
        new webpack.optimize.CommonsChunkPlugin({
            async: true,
            children: true,
            minChunks: 3,
        }),
        new webpack.HotModuleReplacementPlugin(),
        new webpack.NamedModulesPlugin(),
        new HtmlwebpackPlugin({
            template: './src/index_tpl.html',
            filename: 'index.html',
            inject: true,
            hash: true,
        }),
        new webpack.DefinePlugin({
            'process.env': {
                'NODE_ENV': JSON.stringify('dev'),
            },
        }),
    ]
```
-   `CommonsChunkPlugin` 提取公共模块插件。第一项配置多数是把在entry所依赖的库文件提取出来并重新命名。因为为所依赖的package,稳定性高，不常变化，所以不加入hash码，提取出来不必每次重新打包。第二项配置是把所有子chunk的公共依赖（引用次数大于minChunks）提取打包。当使用相应子chunk模块时，它将自动并行下载所打包的公共依赖。([帮助范例](https://github.com/webpack/webpack/issues/5386))
-   `HotModuleReplacementPlugin` webpack HMR 插件配置。就React来说，配合react-hot-loader实现热替换，做到不重新刷新，完成对页面部分结构的替代更新。
-   `NamedModulesPlugin` 显示模块的相对路径，开发环境下方便调试。
-   `HtmlwebpackPlugin` 针对打包后的JS文件，插入到html模板中，完成最后一步的自动化构建。插件默认支持ejs(pug)、html文件格式，其中，inject参数可选，head或者true(等同于body)。默认插入所有output的JS文件；多页情况下，配置chunks参数进行不同页面下的JS引入。
-   `DefinePlugin` 定义变量，完成对生产环境、开发环境下的区分。
