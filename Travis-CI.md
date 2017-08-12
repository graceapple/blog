# Travis-CI
1. Travis是一个在线的CI(Continuous Integration)持续集成构建项目。可以把他想象成一个Web Linux，使用yml语法驱动。对于github的开源项目是免费的。
2. 账户关联

使用Github账户登录 https://travis-ci.org/， 点击Accouts进入用户个人页面，并Sync accout，获取自己的github repository。

官方有教集成步骤

![image](https://raw.githubusercontent.com/icepy/_posts/master/img/sync-repo.png)

3. .travis.yml

只要在项目跟目录下加.traivs.yml文件即可
```
language: node_js
node_js:
          - "4"
before_script:
          - npm install -g mocha
```
以上是一个简单的例子，大致都是高速travis如何准备环境的
4. 遇到的一个问题

在exercise3中，用上述travis配置，并在karma.config.js中配了browser是chrome，本地执行karma start测试通过，在travis上一直报错：不能启动chrome。

上网搜了一下，在.travis.yml中对chrome进行了如下配置
```
language: node_js
node_js:
          - "4"
before_install:
  - export CHROME_BIN=chromium-browser
  - export DISPLAY=:99.0
  - sh -e /etc/init.d/xvfb start
  - sleep 3
before_script:
          - npm install -g mocha
```
- linux上装的chrome是chromeium-browser
- setup一个fake的DISPLAY
- 起一个xvfb（一个fake的GUI 环境）

PS:2，3都不知道为什么要这么设。。但是这么设过之后就work了

5. 直接push触发CI build
## reference
[Travis Doc](https://docs.travis-ci.com/)

[run js test in chrome on Travis](https://swizec.com/blog/how-to-run-javascript-tests-in-chrome-on-travis/swizec/6647)

[Travis + coveralls](https://div.io/topic/1674)

[从Github到Travis](http://www.jianshu.com/p/c80b37f775a0)


