# understanding of style-loader

# __webpack_require__ 的定义

```js
// The module cache
var installedModules = {};
/******/
// The require function
function __webpack_require__(moduleId) {
/******/
	// Check if module is in cache
	if(installedModules[moduleId]) {
		return installedModules[moduleId].exports;
	}
	// Create a new module (and put it into the cache)
	var module = installedModules[moduleId] = {
		i: moduleId,
		l: false,
		exports: {},
		hot: hotCreateModule(moduleId),
		parents: (hotCurrentParentsTemp = hotCurrentParents, hotCurrentParents = [], hotCurrentParentsTemp),
		children: []
	};
/******/
	// Execute the module function
	modules[moduleId].call(module.exports, module, module.exports, hotCreateRequire(moduleId));
/******/
	// Flag the module as loaded
	module.l = true;
/******/
	// Return the exports of the module
	return module.exports;
}
```

# index.css 被 style-loader 处理后的结果

style-loader 是一个 pitch loader, 主要是处理了把 css 添加为页面的 style 标签, 以及处理 css 的 [hot reload](https://github.com/webpack-contrib/style-loader/blob/fc24512b395a8a3fd4074d88cd3f7b195f0bcef2/index.js)

`pitch loader` 与文件的内容无关(从 `module.exports.pitch` 函数只有一个 `request` 参数也能看得出这一点)，所以只与模块 id 有关，所以即使文件变化了, 也不需要修改。这就为 hot reload 的处理提供了一个容器。

> 关于 pitch loader, refer to [the issue](https://github.com/webpack/webpack/issues/360)

下面是 index.css 经过 `style-loader` 处理后生成的模块:

```js
/*!*******************!*\
  !*** ./index.css ***!
  \*******************/
/*! no static exports found */
(function(module, exports, __webpack_require__) {
  var content = __webpack_require__("./node_modules/_css-loader@1.0.1@css-loader/index.js!./index.css");

  var options = {"hmr":true}


  var update = __webpack_require__("./node_modules/_style-loader@0.23.1@style-loader/lib/addStyles.js")(content, options);

  module.hot.accept("./node_modules/_css-loader@1.0.1@css-loader/index.js!./index.css", function() {
		// When the styles change, update the <style> tags
    var newContent = __webpack_require__("./node_modules/_css-loader@1.0.1@css-loader/index.js!./index.css");

    update(newContent);
  });

	// When the module is disposed, remove the <style> tags
  module.hot.dispose(function() { update(); });
})
```

可以看出 `style-loader` 处理后的 `index.css` 的模块，确实和 `index.css` 的内容无关。但是它会去 require 了 `css-loader` 处理的 `index.css`: `./node_modules/_css-loader@1.0.1@css-loader/index.js!./index.css`。其代码如下:

```js
/***/
/*!***************************************************************!*\
  !*** ./node_modules/_css-loader@1.0.1@css-loader!./index.css ***!
  \***************************************************************/
/*! no static exports found */

exports = module.exports = __webpack_require__("./node_modules/_css-loader@1.0.1@css-loader/lib/css-base.js")(false);

exports.push([module.i, "html {\n  width: 100%;\n  height: 100%;\n}\n", ""]);
```

本地开发修改 `index.css` 文件后, `webpack-dev-server` 会向客户端代码发送 `websocket` 通知, 网页端接收到通知后，会将新的 `./node_modules/_css-loader@1.0.1@css-loader!./index.css` 下载到浏览器中, 最后 `webpack` 会 `./index.css` 里的 hot reload 代码会被触发，从而更新页面。


# webpack 打包过程调试.

* chrome 浏览器安装 NIM, 并启用自动模式

![](https://raw.githubusercontent.com/njleonzhang/image-bed/master/assets/e58671ba-5bb4-31f7-6b54-3d0d76759c8f.png)

* 通过 node 命令并带调试参数启动 webpack:
```
 node --inspect-brk  node_modules/webpack-cli/bin/cli.js --config webpack.config.js
```

* NIM 的调试模式会自动打开一个 chrome 窗口，并断点在
