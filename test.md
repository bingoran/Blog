# 构建准备

webpack-cli 的最后，webpack 函数执行完成之后，返回了一个 compiler 实例；

这一篇文章，我们主要看看这个 webpack 函数，到底在编译的过程中起了什么作用；

其实，通过标题，我们大概能猜到，这个 webpack 函数是在为编译阶段做准备；

#### 函数位置

webpack 函数所在的位置，是在 

```
node_modules -> webpack -> lib -> webpack.js
```

注意，webpack 库下面有两个 webpack.js 文件，一个是在

```
 bin -> webpack.js
```

一个在

```
lib -> webpack.js
```

bin ->  webpack.js 这个文件的作用我们在 ` webpack-cil 的职责` 中已经分析过，这里不再赘述；

这里，我们的主角是 lib -> webpack.js；

### webpack.js 构建准备流程图

<img src="http://lstatic.bingoran.com/common/img/1619518526696-074056e0-5ead-4f21-ab29-02e3408e2879.jpeg" alt="img" style="zoom:50%;" />

####  webpack.js 源码分析

webpack.js

```js
...
// 配置文件是数组的时候，创建多个实例，不常用
const createMultiCompiler = (childOptions, options) => {...};

/**
 * 创建真正的 Compiler 实例
 */
const createCompiler = rawOptions => {
	// 整理成为完整的 webpack 配置文件
	const options = getNormalizedWebpackOptions(rawOptions);
	// 设置 context，当前环境就是 node 当前的执行目录 process.cwd()
	applyWebpackOptionsBaseDefaults(options);
	// 新建编译器实例
	const compiler = new Compiler(options.context);
	compiler.options = options; // 配置文件挂载
	// 注册环境
	// 1、监听了 beforeRun 这个钩子，用于构建之前清理缓存
	// 2、定义 inputFileSystem、outputFileSystem、watchFileSystem 等输入输出流
	new NodeEnvironmentPlugin({
		infrastructureLogging: options.infrastructureLogging
	}).apply(compiler);

	// 注册 webpack 配置文件插件
	if (Array.isArray(options.plugins)) {
		for (const plugin of options.plugins) {
			if (typeof plugin === "function") {
				plugin.call(compiler, compiler);
			} else {
				plugin.apply(compiler);
			}
		}
	}
	// 做一些初始的操作，给配置文件做一些设置默认的参数
	applyWebpackOptionsDefaults(options);
	// 初始化环境信息
	compiler.hooks.environment.call(); // 没找到相应的监听，webpack5 留给了开发者自己使用
	compiler.hooks.afterEnvironment.call();
	// 根据所有配置 option 参数转换成对应的 webpack 内部插件
	new WebpackOptionsApply().process(options, compiler);
	compiler.hooks.initialize.call();
	return compiler;
};

const webpack = ((options, callback) => {

		const create = () => {
			validateSchema(webpackOptionsSchema, options);
			let compiler;
			let watch = false;
			let watchOptions;
			// 如果配置文件是数组形式存在多分配置文件，则会创建多个编译器分别处理
			if (Array.isArray(options)) {
				compiler = createMultiCompiler(options, options);
				watch = options.some(options => options.watch);
				watchOptions = options.map(options => options.watchOptions || {});
			} else {
				// 我们的常规处理方式，就是配置文件是一个普通的对象形式
				compiler = createCompiler(options);
        // 监听模式：只有 watch 为 true 的时候 watchOptions 才有意义
				watch = options.watch; 
        // 监听模式的配置参数（不监听的文件夹，刷新频率等）
				watchOptions = options.watchOptions || {}; 
			}
			return { compiler, watch, watchOptions };
		};

		if (callback) {
			try {
				const { compiler, watch, watchOptions } = create();
				if (watch) {
					// 如果是监听模式：走这里
					compiler.watch(watchOptions, callback);
				} else {
					// 打包编译模式走这里
					// run 是编译的入口方法
					compiler.run((err, stats) => {
						compiler.close(err2 => {
							callback(err || err2, stats);
						});
					});
				}
				return compiler;
			} catch (err) {...}
		} else {
			const { compiler, watch } = create();
			...
			return compiler;
		}
	}
);

module.exports = webpack;
```

可以看到，webpack.js 文件里边主要就这 3 个函数

1、webpack

2、createCompiler

3、createMultiCompiler

下面，我们主要分析这 3 个函数

#### webpack 函数

1、create 函数的作用是，根据配置文件，实例化不同的 compiler，并且返回监听模式的相关配置。注意：这里配置文件一般只返回一个对象；对于返回数组的情况极少，所以暂时不看多编译器打包处理。

2、判断 callback 是否为空；如果为空，则只返回 compiler 对象；如果不为空，则根据配置文件按情况实例化 compiler；

3、根据是否是监听模式，再调用 compiler.watch 或者 compiler.run 方法

#### createMultiCompiler 函数

创建多编译器，这个函数只有在配置文件是数组的时候才会使用到，目前没接触过，暂不做分析。

#### createCompiler 函数

1、整理 webpack 配置文件，将部分配置转换成 webpack 需要的格式
2、创建 context 上下文，这里使用的是 process.cwd()，指当前 node 的工作目录
3、创建 compiler 实例
4、初始化 NodeEnvironmentPlugin 插件

* 监听了 beforeRun 这个钩子，用于构建之前清理缓存

* 定义 inputFileSystem、outputFileSystem、watchFileSystem 等输入输出流

5、初始化用户配置的插件，注册插件钩子

```js
if (Array.isArray(options.plugins)) {
		for (const plugin of options.plugins) {
			if (typeof plugin === "function") {
				plugin.call(compiler, compiler);
			} else {
				plugin.apply(compiler);
			}
		}
	}
```

这里可以看到，配置文件里的 plugins 必须是数组才可以正确解析，而且数组里边要么是函数，要么是实例化的插件类。

5、进一步优化 options，给一些配置赋上默认值
6、初始化 webpack 内部插件，例如 js 解析器、缓存插件、添加入口的插件等

```js
// 根据所有配置 options 参数转换成对应的 webpack 内部插件
new WebpackOptionsApply().process(options, compiler);
```

WebpackOptionsApply().process 这个函数比较重要，下面会专门介绍 WebpackOptionsApply

