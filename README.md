## Intro

This is a plugin for [webpack](https://github.com/webpack/webpack).

The main aim is to provide a tool to upload js/css files used in html to cdn, and then replace the reference with the corresponding cdn url.

## Environment requirement

node >= 7.4.0

## Install

```bash
npm i -D webpack-upload-plugin
```

## Notice

This plugin does not provide a service as uploading to cdn.

In fact, it actually depends on such service.

This plugin is for webpack >= 2.

For webpack@4, set `mode` to `'none'`!

This plugin _doesn't_ work well with `UglifyJs` plugin! Use `beforeUpload` if you want to compress anyway.

## Dependency

`webpack-upload-plugin` relies on the existence a `cdn` object with an `upload` method described as below.

```typescript
type cdnUrl = string
interface cdnRes {
  [localPath: string]: cdnUrl
}
// this is what cdn package looks like
interface cdn {
  upload: (localPaths: string[]) => Promise<cdnRes>
}
```

If typescript syntax is unfamiliar, here is another description in vanilla javascript.

```js
/**
 * @param {string[]} localPath: list of paths of local files
 * @return Promise<cdnRes>: resolved Promise with structure like {localPath: cdnUrl}
 */
function upload(localPath) {
  // code
}
const cdn = {
  upload
}
```

## Use case

### Basic one

For a simple project with such structure:

```
+-- src
|   +-- assets
|   |   +-- avatar.png
|   +-- index.js
|   +-- index.css
+-- dist
+-- index.html
+-- webpack.config.js
```

```js
// in webpack.config.js
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const UploadPlugin = require('webpack-upload-plugin')
const cdn = require('xxx-cdn')

module.exports = {
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
    publicPath: ''
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'babel-loader'
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader']
      },
      {
        test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
        loader: 'url-loader',
        options: {
          limit: 10000
        }
      }
    ]
  },
  mode: 'none', // important! important! important!
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].css',
      chunkFilename: '[id].css'
    }),
    new HtmlWebpackPlugin({
      filename: 'index.html',
      template: 'index.html',
      inject: true
    }),
    new UploadPlugin(cdn)
  ]
}
```

> For webpack v3 users, use `extract-text-webpack-plugin` instead of `mini-css-extract-plugin`
>
> For webpack v4 users, you can add missing plugins back manually. See details [here](https://webpack.js.org/concepts/mode/#usage)

### Complex one with Server Template

Run webpack in `build`, then copy all emitted files from `build/dist` to `project/src`.

Public can only access files from `project/public`

```
+-- project
| +-- src
| +-- public
+-- build
| +-- src
| | +-- assets
| | |   +-- avatar.png
| | +-- index.js
| | +-- index.css
| +-- dist
| +-- index.html
| +-- webpack.config.js
```

```js
// only focus on WebpackUploadPlugin here
{
  plugins: [
    new UploadPlugin(cdn, {
      src: path.resolve(__dirname, '..', 'project/src'),
      dist: path.resolve(__dirname, '..', 'project/public'),
      staticDir: path.resolve(__dirname, '..', 'project/src'),
      dirtyCheck: true
    })
  ]
}
```

> Make sure `WebpackUploadPlugin` is after any copy-related plugins in `plugins` field.

> If in `project/public`, there are different prefix from `publicPath` you passed to webpack, then use `replaceFn` to remove such prefix.

```js
const config = {
  replaceFn(content, location) {
    return path.extname(location) === '.html'
      ? content.replace(prefix, '')
      : content
  }
}
```

> If the copy process takes a long time, use `waitFor` to make sure only start uploading when things are settled.

## Configuration

In webpack.config.js

```js
const WebpackUploadPlugin = require('webpack-upload-plugin')
const cdn = require('some-cdn-package')
module.exports = {
  plugins: [new WebpackUploadPlugin(cdn, option)]
}
```

`option` is optional.

Valid fields for `option` are showed below:

### [`src`]: string

Where your valid raw template files would appear (with reference to local js/css files). Default to be where html files would be emitted to based on your webpack configuration.

For templates not dynamically emitted by webpack, eg, files like `php`, `tpl`, `phtml` and so on that are managed by some server logic not client one, this field could be used.

> Use _absolute_ path

### [`dist`]: string

Where to emit final template files. Only use this when there is a need to separate origin outputs with cdn ones. Default to be same as `src`.

Think in this way:

template from src -> template with cdn reference -> template to dist

Use `dist` only if the third step is needed.

> Use _absolute_ path

### [`urlCb`]: (cdnUrl: string, localPath: string) => string

Adjust cdn url accordingly. Cdn url would be passed in, and you need to return a string.

```js
const url = 'http://domain.com/cdn/bundle.js'
const urlCb = input => input.replace(/^https?/, 'https')
```

### [`resolve`]: string[]

Type of templates needed to match. In case you have a project with php, smarty, or other template language instead of html. Default to `['html']`

Again, for projects using server language template, like `php`, you could set `resolve` to `['php', 'phtml']`

### [`staticDir`]: string | string[]

If static files (js/css/images etc) emitted by webpack is not what you want, or not enough(normally happens when you need to copy all resources emitted by webpack to another directory), then set `staticDir` to the directory that contains all your desired resource files (js/css/images etc).

> Use _absolute_ path

### [`beforeUpload`]: (fileContent: string, fileLocation: string) => string

_Compression_ can be done here. Two arguments are `fileContent` and `fileLocation` (with extension name of course). You need to return the compression result as string.

```js
// if you want to compress js before upload
const UglifyJs = require('uglify-js')
const path = require('path')
const beforeUpload = (content, location) => {
  if (path.extname(location) === '.js') {
    return UglifyJs.minify(content).code
  }
  return content
}
```

### [`replaceFn`]: (fileContent: string, location: string) => string

For some complex (ancient) projects, you may have multiple `publicPath` or corresponding concepts. To handle such cases accordingly, you can pass a `replaceFn` function, which will receive two parameters, which are `fileContent` and `location` in that order. `fileContent` would be file in string format with local resources reference. `location` is the location of `fileContent` on your file system. This function will be called when plugin start to replace local reference. The string `replaceFn` returns will represent the desired `publicPath`, which will be used as the input template to replace all local reference with cdn ones.

```js
// in your latest webpack.config.js
module.exports = {
  output {
    publicPath: 'public/static'
  }
}
```

```html
<!-- in an ancient template file -->
<!-- bundle.js is actually emitted by webpack -->
<!-- but some copy action involved in later sometime -->
<!-- public/static/bundle.js -> public/js/bundle.js -->
<!-- but this src attribute cannot be changed due to some weird reason -->
<script src="public/js/bundle.js"></script>
```

In such case, `staticDir` could be useful. Or you can use `replaceFn` as followed:

```js
const replaceFn = (content, location) => {
  const oldPublicReg = /src="public\/js/
  if (oldPublicReg.test(content)) {
    return content.replace(oldPublicReg, `src="public/static"`)
  }
}
```

In this way, `public/static/bundle.js` will be uploaded before it get moved to `public/js`, and the reference replacement stay accurate.

### [`waitFor`]: () => Promise<\*>

A function that returns a Promise. The plugin will wait for the Promise to resolve and then start everything.

```js
// things won't start till 1000ms later
const waitFor = new Promise(resolve => {
  setTimeout(resolve, 1000)
})
```

### [`dirtyCheck`]: boolean

For cases where chunk file can also be entry file, set `dirtyCheck` to `true` to make sure entry file would be updated properly.

### [`onFinish`]: () => any

Called when everything finished. You can further play with files here.

```js
const onFinish = () => {
  console.log('just want to print this')
}
```

### [`onError`]: (e: Error) => any

Called when encounter any error.

### [`logLocalFiles`]: boolean

Whether to print all uploading file names during the process

### [`passToCdn`]: object

Extra config to pass to `cdn.upload` method. Something Like `cdn.upload(location, passToCdn)`.

```js
// if your original cdn package has API like this
const cdn = require('some-cdn-package')
const passToCdn = {
  https: true
}
cdn.upload(files, passToCdn)
```

### [`enableCache`]: boolean

Enable cache to speed up. Default to `false`.

### [`cacheLocation`]: string

Directory to emit the upload cache file. Use this when you want to manage the cache file by any VCS.

```js
const path = require('path')
const cacheLocation = path.resolve(__dirname, 'cacheDirectory')
```

### [`sliceLimit`]: number

Uploading files is not done by once. By using `sliceLimit`, you can limit the number of files being uploaded at once.

> `src` and `dist` work best with absolute path!
>
> Pay extra attention to your `publicPath` field of `webpack.config.js`, `''` is likely the best choice.

Viola! That's all : )

## License

[MIT](http://opensource.org/licenses/MIT)

Copyright (c) 2017-present, Yuchen Liu
