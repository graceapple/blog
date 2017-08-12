# Unit test&coverage for Github open source project:Travis + coveralls + mocha + istanbul
## coverage是什么？
test coverage是对要进行测试的源代码测试程度的表示。100%的coverage表示测试用例跑到了每一行源代码。
## what you need?
- test runner: 真正执行测试用例的框架(mocha)。
- code coverage tool： 生成coverage report的工具(istanbul)
- coverage insight tool： 对coverage report做数据可视化的工具(coveralls.io)
## steps
1. 用github账户登录https://www.coveralls.io, 并active需要做coverage的repository
2. 用github账户登录Travis，sync repository，配置repository与travis集成
3. 安装npm包
```
$ npm i istanbul mocha coveralls mocha-lcov-reporter -D
```
4. 配置 npm script

主要配置两个命令
 - npm run test:用于启动测试执行，并生成coverage报告
 - npm run coveralls: Travis CI在执行完test之后，会执行这个命令把生成的coverage report发送给coveralls。
```
  "scripts": {
    "test": "./node_modules/istanbul/lib/cli.js cover ./node_modules/mocha/bin/_mocha --report lcovonly -- -R spec",
    "coveralls": "cat ./coverage/lcov.info | ./node_modules/coveralls/bin/coveralls.js && rm -rf ./coverage"
  },

```
其中，使用_mocha而不是mocha，因为网上说，_mocha是真正的test runner，mocha是对_mocha的wrapper， 并且直接执行mocha会起一个子进程，导致istanbul报错。
```
No coverage information was collected, exit without writing coverage information
```
具体我没有试过，直接用的_mocha。

如果想在本地执行run coverage，需要加一个.coverall.yml文件。把repo_token写进去，repo_token可以在coveralls.io的settings中得到。用于标识每个repository。
5. 配置CI的.travis.yml
让travis在执行完npm run test之后，执行npm run coveralls
```
after_success: 'npm run coveralls'
```
6. 给readme加badge
想在readme里有build 100%的徽章，在readme中加入如下代码
```
[![Coverage Status](https://coveralls.io/repos/<account>/<repository>/badge.svg?branch=master)](https://coveralls.io/r/<account>/<repository>?branch=master)

```
把accout和repository改成自己的项目。
## reference
[coveralls doc](https://github.com/nickmerwin/node-coveralls)

[how to add istanbul code coverage badge to your github project](http://maximilianschmitt.me/posts/istanbul-code-coverage-badge-github/)

[js code coverage with istanbul and coveralls](http://codeheaven.io/javascript-code-coverage-with-instanbul-and-coveralls/)

[node+mocha+travis+istanbul+coveralls](http://dsernst.com/2015/09/02/node-mocha-travis-istanbul-coveralls-unit-tests-coverage-for-your-open-source-project/)

[Travis CI + Coveralls](https://div.io/topic/1674)


