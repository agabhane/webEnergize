---
title: Getting away from bower using gulp and browserify
date: 2018-01-12 17:05:49
categories:
- angularjs
tags:
- angularjs
- gulp
- browserify
---
# Introduction

Recently I have moved my angularjs project from bower to npm. There were two reasons for that -
* Bower has started suggesting to use yarn or npm
* Not all packages are available on bower.

So I thought, its good time to move away from bower and start using npm as new package manager for my project. But moving away from bower and using npm is not straight forward and thats the motivation behind this post.

# Application setup

For the sake of simplicity, we will be developing a very small angularjs application which displays current time. As we go along, I will be explaining why we need bowserify.
Full source code can be found at [gulp-browserify-setup-angularjs](https://github.com/agabhane/gulp-browserify-setup-angularjs)

**Folder Stucture**
`app` folder will host our angularjs modules and controllers.
```
src
  -app
  -index.html
  -vendor.js
package.json
gulpfile.js
```

**package.json**
You can start with `npm init` and add below dependencies.
Use `npm install` to install all the dependencies.

{% codeblock lang:javascript %}
"devDependencies": {
  "browserify": "^14.5.0",             // browserify
  "del": "^3.0.0",                     // del folder or files
  "gulp": "^3.9.1",                    // gulp task runner
  "gulp-angular-filesort": "^1.1.1",   // gulp plugin to sort angularjs file as per their dependencies 
  "gulp-inject": "^4.3.0",             // gulp plugin to inject js or css files to index.html
  "gulp-load-plugins": "^1.5.0",       // load gulp plugins without writting require statment for each and every plugin
  "gulp-sourcemaps": "^2.6.2",         // generate souremap
  "gulp-uglify": "^3.0.0",             // uglify code
  "run-sequence": "^2.2.0",            // running gulp tasks in sequence
  "vinyl-buffer": "^1.0.1",            // convert stream to buffer
  "vinyl-source-stream": "^2.0.0"      // read stream
},
"dependencies": {
  "angular": "^1.6.8",                 // angularjs
  "moment": "^2.20.1"                  // momentjs
}
{% endcodeblock %}
For such a small application we don't need momentjs. But to showcase more than one vendor library, we will be using momentjs.

# Code

**index.html**
`src/index.html` file will have module and controller bindings along with placeholders to inject *vendor* and *source* scripts.
If you look at the html file, we are not adding any script right now. It will be injected by `gulp-inject` plugin.
{% codeblock lang:html %}
<html>
  <head>
    <title>Getting away from bower using gulp and browserify</title>
  </head>
  <body>
    <div ng-app="myapp" ng-controller="myctrl">
      <div style="text-align:center; font-size:30px;" ng-bind="currentTime"></div>
    </div>

    <!-- vendor:js -->
    <!-- endinject -->
    <!-- inject:js -->
    <!-- endinject -->
  </body>
</html>
{% endcodeblock %}

**mymodule.js**
`src/app/mymodule.js` file will have angularjs module `myapp`.
{% codeblock lang:javascript %}
angular.module('myapp', []);
{% endcodeblock %}

**mycontroller.js**
`src/app/mycontroller.js` file will have controller `myctrl`.
We have injected `$interval` service to update our time after every 1 second.
{% codeblock lang:javascript %}
angular.module('myapp')
.controller('myctrl', ['$scope', '$interval', function($scope, $interval){
  $scope.currentTime = moment().format('DD MMM YYYY HH:mm:ss');
  $interval(function(){
    $scope.currentTime = moment().format('DD MMM YYYY HH:mm:ss');
  },1000);
}]);
{% endcodeblock %}

**vendor.js**
Since we are not adding any scripts directly to out `index.html` file, we will be adding our dependencies in `src/vendor.js` file.
Here `angular` and `moments` are being searched from `node_modules/` [using node's module lookup algorithm](https://github.com/browserify/resolve).
{% codeblock lang:javascript %}
window.angular = require('angular');
window.moment = require('moment');
{% endcodeblock %}

Browsers dont't understand `require`, and thats why we need browserify here.

{% blockquote @ http://browserify.org/ Browserify %}
Browserify lets you require('modules') in the browser by bundling up all of your dependencies.
{% endblockquote %}

# Gulp
[Gulp](https://gulpjs.com/) is a javascript task runner same as [Grunt](https://gruntjs.com/).
Gulp is more of code based unlike config based Grunt.

We are going to few tasks in gulp. Create new file `gulpfile.js` in our root folder.
Full source code can be found at [gulp-browserify-setup-angularjs/gulpfile.js](https://github.com/agabhane/gulp-browserify-setup-angularjs/blob/master/gulpfile.js)

**task 'build-vendor'**
`build-vendor` task will take `src/vendor.js` file as input and bundle all dependencies into `dist/vendor.js` from a recursive walk of the `require()` graph.
Basically it concatinates `angular` and `moment` code into `dist/vendor.js` file.

{% codeblock lang:javascript %}
gulp.task('build-vendor', function () {
  // set up the browserify instance on a task basis
  var b = browserify({
    entries: './src/vendor.js',                   // vendor source file
    debug: true
  });

  return b.bundle()
    .pipe(source('vendor.js'))
    .pipe(buffer())
    .pipe(sourcemaps.init({loadMaps: true}))      
        .pipe(uglify())                           // uglify code 
    .pipe(sourcemaps.write('./'))                 // write sourcemap
    .pipe(gulp.dest('./dist/js/'));
});
{% endcodeblock %}

**task 'build-js'**
`build-js` task will copy all `.js` files in `app` folder to `dist/js/`.

{% codeblock lang:javascript %}
gulp.task('build-js', function () {
  return gulp.src('./src/app/**/*.js')
    .pipe(gulp.dest('./dist/js/'));
});
{% endcodeblock %}

**task 'build-html'**
Copy `index.html` to `dist`.
{% codeblock lang:javascript %}
gulp.task('build-html', function () {
  return gulp.src('src/index.html')
    .pipe(gulp.dest('dist'));
});
{% endcodeblock %}

**task 'inject'**
Now that we have `vendor.js` and all source code `.js` files copied to `dist`, we need to inject them in our `dist/index.html` file.

{% codeblock lang:javascript %}
gulp.task('inject', function () {
  var target = gulp.src('dist/index.html');
  var vsources = gulp.src(['dist/js/vendor.js'], { read: false });
  var msources = gulp.src(['dist/**/*.js', '!dist/js/vendor.js']);
  return target.pipe($.inject(msources.pipe($.angularFilesort()), { relative: true }))
    .pipe($.inject(vsources, { relative: true, name: 'vendor' }))
    .pipe(gulp.dest('dist'));
});
{% endcodeblock %}

And finally write one task to club all of them together.
{% codeblock lang:javascript %}
gulp.task('build', function (done) {
  sequence('clean:dist', ['build-vendor', 'build-js', 'build-html'], 'inject', done);
});
{% endcodeblock %}

Run `gulp build`.
This will build you code in `dist` folder.
You can use your favourite http server to test. I am using [http-server](https://www.npmjs.com/package/http-server).
```
npm install http-server
http-server dist/
```

# References
* https://github.com/browserify/browserify#usage
* https://github.com/gulpjs/gulp/blob/master/docs/recipes/browserify-uglify-sourcemap.md
