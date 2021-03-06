# [gulp](https://github.com/gulpjs/gulp)-watch [![NPM version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url] [![Coverage Status](https://coveralls.io/repos/floatdrop/gulp-watch/badge.png)](https://coveralls.io/r/floatdrop/gulp-watch) [![Dependency Status][depstat-image]][depstat-url]
> Watch, that actually is an endless stream

This is implementation of [`gulp.watch`](https://github.com/gulpjs/gulp/blob/master/docs/API.md#gulpwatchglob--opts-cb) with endless stream approach. If `gulp.watch` is working for you - stick with it, otherwise you can try `gulp-watch` plugin.

Main reasons of `gulp-watch` existance is that it can easly (with a little help of [`gulp-plumber`](https://github.com/floatdrop/gulp-plumber) achieve per-file rebuilding on file change:

![Awesome demonstration](https://github.com/floatdrop/gulp-watch/raw/master/img/2014-01-09.gif)

## Usage

### Batching mode

This is close to bundled `gulp.watch`, but with some tweaks. First - files will be grouped by timeout of `200` and passed into stream inside callback (this will keep `git checkout` commands do rebuilding only once). Second - callbacks will __never__ run parallel (unless you remove `return`), until one stream ends working.

```js
var gulp = require('gulp'),
    watch = require('gulp-watch');

gulp.task('default', function () {
    gulp.src('scss/**/*.scss')
        .pipe(watch(function(files) {
            return files.pipe(sass())
                .pipe(gulp.dest('./dist/'));
        }));
});
```

If you want to watch all directories, include those, which will be __created__ after:

```js
var gulp = require('gulp'),
    watch = require('gulp-watch');

gulp.task('default', function () {
    watch({glob: 'scss/**/*.scss'}, function(files) {
        return files.pipe(sass())
            .pipe(gulp.dest('./dist/'));
    });
});
```

### Continuous stream of events

This is usefull, when you want blazingly fast rebuilding per-file.

__Be aware:__ `end` event is never happens in this mode, so plugins dependent on it will never print or do whatever they should do on `end` task.

```js
// npm i gulp gulp-watch gulp-sass

var gulp = require('gulp'),
    watch = require('gulp-watch'),
    plumber = require('gulp-plumber'),
    sass = require('gulp-sass');

gulp.task('default', function () {
    gulp.src('scss/**', { read: false })
        .pipe(watch())
        .pipe(plumber()) // This will keeps pipes working after error event
        .pipe(sass())
        .pipe(gulp.dest('./dist/'));
});
```

If you want to watch all directories, include those, which will be __created__ after:

```js
gulp.task('default', function () {
    watch({ glob: 'sass/**/*.scss' })
        .pipe(plumber())
        .pipe(sass())
        .pipe(gulp.dest('./dist/'));
});
```

### Trigger for mocha

[Problem with `gulp.watch`](https://github.com/gulpjs/gulp/issues/80) is that will run your test suit on every changed file per once. To avoid this [`gulp-batch`](https://github.com/floatdrop/gulp-batch) was written first, but after some time it became clear, that `gulp.watch` should be a plugin with event batching abilities.

```js
var grep = require('gulp-grep-stream');
var mocha = require('gulp-mocha');
var plumber = require('gulp-plumber');

gulp.task('watch', function() {
    gulp.src(['lib/**', 'test/**'], { read: false })
        .pipe(watch({ emit: 'all' }, function(files) {
            files
                .pipe(grep('*/test/*.js'))
                .pipe(mocha({ reporter: 'spec' }))
                .on('error', function() {
                    if (!/tests? failed/.test(err.stack)) {
                        console.log(err.stack);
                    }
                })
        }));
});

gulp.task('default', function () {
    gulp.run('watch');
});

// run `gulp watch` or just `gulp` for watching and rerunning tests
```

## API

### watch([options, callback])

This function creates have two different modes, that are depends on have you provice callback function, or not. If you do - you get __batched__ mode, if you not - you get __stream__.

### Callback signature: `function(events, [done])`

 * `events` - is `Stream` of incoming events.
 * `done` - is callback for your function signal to batch, that you are done. This allows to run your callback as soon as previous end.

### Options:

This object passed to [`gaze` options](https://github.com/shama/gaze#properties) directly, so see documentation there. For __batched__ mode we are using [`gulp-batch`](https://github.com/floatdrop/gulp-batch#api), so options from there are available. And of course options for [`gulp.src`](https://github.com/gulpjs/gulp#gulpsrcglobs-options) used too. If you do not want content from watch, then add `read: false` to options object.

#### options.emit
Type: `String`
Default: `one`

This options defines emit strategy:

 * `one` - emit only changed file
 * `all` - emit all watched files (and folders), when one changes

#### options.passThrough
Type: `Boolean`  
Default: `true`

This options will pass vinyl objects, that was piped into `watch` to next Stream in pipeline.

#### options.glob
Type: `String|Array`  
Default: `undefined`

If you want to detect new files, then you have to use this option. When `gulp-watch` gets files from `gulp.src` it looses the information about pattern of matching - therefore it can not detect new files.

#### options.base
Type: `String`  
Default: `undefined`

Use explicit base path for files from glob.

#### options.emitOnGlob
Type: `Boolean`  
Default: `true`

If `options.glob` is used, gulp-watch, by default, will emit files when beginning to watch them -- much like `gulp.src()`. Otherwise, disable this option.

Example:
```js
// gulp-watch will not emit like gulp.src(...)
watch({glob:'./src/**/*.md', emitOnGlob: false})
    .pipe(plumber())
    .pipe(anotherPlugin(opts))
    .pipe(gulp.dest('./html'))
```

#### options.name
Type: `String`  
Default: `undefined`

Name of the watcher. If it present in options, you will get more readable output:

![Naming watchers](https://github.com/floatdrop/gulp-watch/raw/master/img/naming.png)

#### options.verbose
Type: `Boolean`  
Default: `false`

This options will enable more verbose output (useful for debugging).

### Methods

Returned Stream from constructor have some useful methods:

 * `close()` - calling `gaze.close` and emitting `end`, after `gaze.close` is done.

### Events

 * `end` - all files are stop being watched.
 * `ready` - just re-emitted event from `gaze`.
 * `error` - when something happened inside callback, you will get notified.

### Properties

 * `gaze` - instance of `gaze` in case you want to call it methods (for example `remove`). Be aware __no one guarantee you nothing__ after you hacked on `gaze`.

### Returns

Stream, that handles `gulp.src` piping.

# License

MIT (c) 2013 Vsevolod Strukchinsky (floatdrop@gmail.com)

[npm-url]: https://npmjs.org/package/gulp-watch
[npm-image]: https://badge.fury.io/js/gulp-watch.png

[travis-url]: http://travis-ci.org/floatdrop/gulp-watch
[travis-image]: https://travis-ci.org/floatdrop/gulp-watch.png?branch=master

[depstat-url]: https://david-dm.org/floatdrop/gulp-watch
[depstat-image]: https://david-dm.org/floatdrop/gulp-watch.png?theme=shields.io
