# karam + mocha + should
测试组件选取karma为测试管理工具，mocha为测试库，should为断言库.
- karma：Karma是一个基于Node.js的JavaScript测试执行过程管理工具（Test Runner）。该工具可用于测试所有主流Web浏览器，也可集成到CI（Continuous integration）工具，也可和其他代码编辑器一起使用。这个测试工具的一个强大特性就是，它可以监控(Watch)文件的变化，然后自行执行，通过console.log显示测试结果。
- mocha：mocha是一个基于nodejs和浏览器集合的各种特性的javascript测试库，并且让异步测试变得简单，支持TDD(测试驱动开发)和BDD(行为驱动开发)，在测试中捕获到异常时，会给出灵活准确的报告。
- should：should是一个基于nodejs的断言库，并且完美支持各种主流的JavaScript测试框架。

## karma
karma 是一个提升测试效率的工具，帮助开发者更好更快速地在多种环境下执行测试代码，拿到测试结果。在运行的时候，它会自动启动配置好的浏览器，同时也会启动一个 node 服务器，然后在启动好的浏览器中执行测试代码，并将测试代码执行结果传回给 node 服务器，然后 node 服务器在打印出收到的执行结果。

一般在packet.json的devdependency里写了karma，直接npm install就可以把karma安装上。根据要使用的浏览器和第三方测试框架，预装合适的插件。
```
"devDependencies": {
    "karma": "^1.7.0",
    "karma-chrome-launcher": "^2.2.0",
    "karma-mocha": "^1.3.0",
    "mocha": "^3.5.0",
    "should": "^11.2.1"
  }
```
但是为了使用方便，会在全局下安装一下karma-cli
```
npm install karam-cli -g
```
直接执行 karma init，根据需要做一些初始化配置，会在项目根目录下，生成一个karma.config.js文件，之后如果要改karma配置，就在此文件中修改。
```
module.exports = function(config) {
  config.set({

    // base path that will be used to resolve all patterns (eg. files, exclude)
    basePath: '',


    // frameworks to use
    // available frameworks: https://npmjs.org/browse/keyword/karma-adapter
    frameworks: ['mocha'],


    // list of files / patterns to load in the browser
    files: [
      'https://cdn.bootcss.com/jquery/2.2.4/jquery.js',
      'node_modules/should/should.js',
      'test/**.js'
    ],


    // list of files to exclude
    exclude: [
    ],


    // preprocess matching files before serving them to the browser
    // available preprocessors: https://npmjs.org/browse/keyword/karma-preprocessor
    preprocessors: {
    },


    // test results reporter to use
    // possible values: 'dots', 'progress'
    // available reporters: https://npmjs.org/browse/keyword/karma-reporter
    reporters: ['progress'],


    // web server port
    port: 9876,


    // enable / disable colors in the output (reporters and logs)
    colors: true,


    // level of logging
    // possible values: config.LOG_DISABLE || config.LOG_ERROR || config.LOG_WARN || config.LOG_INFO || config.LOG_DEBUG
    logLevel: config.LOG_INFO,


    // enable / disable watching file and executing tests whenever any file changes
    autoWatch: true,


    // start these browsers
    // available browser launchers: https://npmjs.org/browse/keyword/karma-launcher
    browsers: ['Chrome'],


    // Continuous Integration mode
    // if true, Karma captures browsers, runs the tests and exits
    singleRun: true,

    // Concurrency level
    // how many browser should be started simultaneous
    concurrency: Infinity
  })
}

```

执行karma start，开始执行测试任务。packet.json里的script中的test命令，配置为karma test，这样在CI集成时，会自动调用npm run test执行开始测试。
```
"scripts": {
    "test": "karma start"
  },
```
karma可以集成第三方测试框架，例如mocha，在karma.config.js中的framework可以配置。上例中配了mocha为framework，karma start会自动调用mocha执行测试代码。

## mocha
mocha也是一个测试框架，适合用在nodejs环境中。和karma相比，缺少流程控制。可以说karma偏重测试流程管理，控制测试流程，在测试完成时，自动生成报告，本地文件有变化时提供增量加载，调用第三方测试框架执行测试，并与CI集成等。mocha才是其中真正执行测试代码的test runner。

为了本地执行命令行方便，一般也会把mocha装在全局下。
```
npm install mocha -g
```
测试集，以函数describe(string, function)封装；测试用例，以it(string, function)函数封装，它包含2个参数；断言，以assert语句表示，返回true或false。另外，mocha在完成异步测试用例时通过done()来标记。
```
var assert = require("assert")
describe('assert', function () {
  it('a和b应当深度相等', function () {
    var a = {
      c: {
        e: 1
      }
    }
    var b = {
      c: {
        e: 1
      }
    }
    // 修改下面代码使得满足测试描述
    assert.deepEqual(a, b)
  })

  it('可以捕获并验证函数fn的错误', function () {
    function fn() {
      xxx;
    }
    // 修改下面代码使得满足测试描述
    assert.throws(fn, Error)
  })
})
```
使用mocha很简单，mocha默认运行项目根目录下test子目录里面的测试脚本。所以，一般都会把测试脚本放在test目录里面，然后执行mocha就不需要参数了。
直接对上面的代码运行mocha，得到下面结果
```
  Array
    #indexOf()
      ✓ should return -1 when the value is not present

  assert
    ✓ a和b应当深度相等
    ✓ 可以捕获并验证函数fn的错误
```

mocha命令行参数除了直接在命令行中指定，还可以写在test目录下的mocha.ops文件中。例如，在mocha.ops中加入：
```
anotherTestDir
--reporter tap
--recursive
```
相当于执行
```
mocha anohterTestDir --reporter tap --recursive
```
## should
使用should作为断言库

在mocha.ops中直接require should，就可以直接在test js文件中直接使用should，而不用require("should")了。
```
--require should
```


## refernence
[karma 测试框架的前世今生](http://taobaofed.org/blog/2016/01/08/karma-origin/)

[node assert API](http://nodejs.cn/api/assert.html)

[Mocha实例教程](http://www.ruanyifeng.com/blog/2015/12/a-mocha-tutorial-of-examples.html)

[前端自动化测试解决方案探析](http://www.imweb.io/topic/5833d14cf8a1d5546059a301)

[karma框架的前世今生](http://taobaofed.org/blog/2016/01/08/karma-origin/)

[karma入门（结合requireJs）](https://segmentfault.com/a/1190000005848842)

[shouldJs](http://shouldjs.github.io/)