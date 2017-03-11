---
title: '[Gulp Workflow] 在提交代码前使用jslint进行代码检查'
date: 2016-04-06 15:29:00
categories: ['工具, 库与框架', Gulp]
tags: [Gulp, jslint, 代码检查]
---

查阅到的解决方案主要还是使用gulp结合git hook来执行pre-commit脚本。比如：

[这里](https://www.qianduan.net/task-master-gulp-lint/)

其实不需要那么麻烦，直接使用gulp的插件即可帮助你生成需要的pre-commit脚本：

### 1. 安装[git-pre-commit](https://www.npmjs.com/package/git-pre-commit)以及[gulp-jshint](https://www.npmjs.com/package/gulp-jshint)

```
npm install git-pre-commit --save-dev
```

<!-- More -->

### 2. 编写jslint任务

```
gulp.task('lint', function() {
  return gulp.src('./www/js/**/*.js')
    .pipe(jshint())
    .on('error', handleError)
    .pipe(jshint.reporter('jshint-stylish'))
    .pipe(jshint.reporter('fail'));
});

function handleError(err){
  'use strict';
  console.log(err.toString());
}
```

上述代码中：
```
pipe(jshint.reporter('fail'))
```

十分重要，它会在jslint发现代码不符合规范时让当前的任务失败。如果这行代码，即使jslint检查出来了错误，最后的git commit也会提交。


### 3. 编写jslint任务

在package.json中，添加下面一行代码：

```
"precommit": "gulp pre-commit"
```

并且在gulpfile.js中，添加相应的任务：

```
gulp.task('pre-commit', ['lint']);
```

上面任务的命名不一定需要是pre-commit，只要保持package.json和gulpfile.js中任务名称一致即可。

### 4. 运行git commit测试

可以故意在某个源码中添加一个分号，然后跑git commit：

```
[15:27:40] Starting 'lint'...

www/js/filters/flagFilters.js
  line 37  col 5  Unreachable ';' after 'return'.
  line 37  col 5  Unnecessary semicolon.

  ⚠  2 warnings

[15:27:41] 'lint' errored after 1.28 s
[15:27:41] Error in plugin 'gulp-jshint'
```

在项目的git/hooks目录下，你可以发现已经帮你生成了pre-commit脚本：

```
lrwxr-xr-x   1 username  staff    75  4  6 14:26 pre-commit -> /path-to-app/node_modules/git-pre-commit/scripts/pre-commit
lrwxr-xr-x   1 username  staff    78  4  6 14:26 pre-commit.js -> /path-to-app/node_modules/git-pre-commit/scripts/pre-commit.js
```

有了这层保障，妈妈再也不用担心我的repo中会有不符合规范的代码啦。




