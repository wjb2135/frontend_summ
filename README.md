# gulp构建之 “前端静态资源的版本管理”
以前的传统方案：
后端 将前端静态资源路径作为参数带入某个处理方法中，然后读取真个文件内容。文件名追加md5，输出到指定目录（runtime/statics根目录下）。
弊端：前端更改某个文件（js/css/...）,如果不做任何操作去刷新页面，蛋痛的事情发生了，更改的代码没生效，页面缓存了上一次已完成加载的静态资源。怎么办？手动去后端指定目录去手动删除生成的文件，解除缓存。

我们需要的解决方案:
在构建阶段计算静态资源的hash值，并将该值已参数的形式追加到```<link>、 <script>```中的url
当某个文件被修改，在构建阶段只会重新计算该文件的hash值而不改变其他文件,并且会自动修改引用了该文件的各个模块的view文件。这样带来的好处：在更新时，非暴力清除所有静态资源文件，充分利用缓存；真正意义上的版本管理。

### 默认方案
使用gulp-rev + gulp-rev-collector，那么默认的效果如下：
``` bash
"css/main.css" => "dist/css/main-3d23dfgdf.css"
"js/main.js" => "dist/js/main-3d23dfgdf.js"
"images/banner.jpg" => "dist/images/banner-3d23dfgdf.jpg"
```
但是如果使用上面的方法，实现的是非覆盖式更新：即每次修改文件后，生成带新版本号的文件，而带旧版本号的文件不会被删除，长此以往，最后发现文件夹里累积了大量带旧版本号的无用文件。比如：main.css经历了三次修改，会累积三个文件：
```basename
main-232hwewef.css
main-234ft4545.css
main-5656gdesf.css
```
### 改进方案
##### 我们期望的效果：
```bash
href="dist/css/main-3d23dfgdf.css" => href="dist/css/main.css?v=3d23dfgdf"
src="dist/js/main-3d23dfgdf.js" => src="dist/js/main.js?v=3d23dfgdf"
src="dist/images/banner-3d23dfgdf.jpg" => src="dist/images/banner.jpg?v=3d23dfgdf"
```
##### 原理：
1、 gulp-rev 根据文件内容计算生成一个hash字符串，如：3d23dfgdf
2、 gulp-rev 生成一个映射表，列出映射关系
```bash
{
    main.css: main.css?v=3d23dfgdf
}
```
3、 gulp-rev-collector根据映射关系表，将<link>、<script>url中的文件名进行替换。
例如：main.css替换成main.css?v=3d23dfgdf
##### 下面开始改进
需要对gulp-rev和gulp-rev-collector进行修改。修改如下：
修改映射关系表中的属性值的格式：
打开node_modules\gulp-rev\index.js
```bash
第144行 manifest[originalFile] = revisionedFile;
改成 manifest[originalFile] = originalFile + '?v=' + file.revHash;
```
修改生成文件的文件名（原来是将hash值直接加入到文件名中，现在需要保持文件名不变）
打开node_modules\rev-path\index.js
```bash
第10行 return filename + '-' + hash + ext;
改成 return filename + ext;
```
打开node_modules\gulp-rev-collector\index.js（gulp-rev-collector的版本指定1.0.0）
```bash
第30行 if ( path.basename(json[key]).replace(new RegExp( opts.revSuffix ), '' ) !==  path.basename(key) ) {
改成 if ( path.basename(json[key]).split('?')[0] !== path.basename(key) ) {
```
避免引用 URL 中的版本号累积：dist/css/main.css?v=3d23dfgdf => dist/css/main.css?v=3d23dfgdf?v=6e4cfcf9de
打开node_modules\gulp-rev-collector\index.js
```bash
第94行 regexp: new RegExp(  dirRule.dirRX + pattern, 'g' ),
改成 regexp: new RegExp(  dirRule.dirRX + pattern+'(\\?v=\\w{10})?', 'g' ),
```
### gulpfile.js配置
```
const gulp = require('gulp');
const babel = require('gulp-babel');
const uglify = require('gulp-uglify');
const rename = require('gulp-rename');
const concat = require('gulp-concat');
const less = require('gulp-less');
const minifycss = require('gulp-minify-css');
const imagemin = require('gulp-imagemin');
const plumber = require('gulp-plumber');

const rev = require('gulp-rev');
const revCollector = require('gulp-rev-collector');

// 压缩图片
gulp.task('convertImg', function() {
  return gulp.src('src/images/*')
    .pipe(imagemin({
        progressive: true,
        svgoPlugins: [{removeViewBox: false}]
    }))
    .pipe(gulp.dest('dist/statics/images/'))
});

//转换less文件   
gulp.task('convertLess', function() {
    return gulp.src('src/less/*.less')//该任务针对的文件
        .pipe(plumber())
        .pipe(less())//该任务调用的模块
        .pipe(concat('main.css'))
        .pipe(gulp.dest('dist/statics/css'))
        .pipe(rename({ suffix: '.min' }))
        .pipe(minifycss())
        .pipe(gulp.dest('dist/statics/css'))
});

//压缩，合并 模块js
gulp.task('convertPageJS', function() {
    return gulp.src('src/js/modules/*.js')      //需要操作的文件
        .pipe(babel({
            presets: ['es2015']
        }))
        .pipe(concat('main.js'))    //合并所有js到main.js
        .pipe(gulp.dest('dist/statics/js'))       //输出到文件夹
        .pipe(rename({suffix: '.min'}))   //rename压缩后的文件名
        .pipe(uglify())    //压缩
        .pipe(gulp.dest('dist/statics/js'))  //输出
});

//压缩，合并 公用js
gulp.task('convertPublicJs', function() {
    return gulp.src('src/js/public/*.js')      //需要操作的文件
        .pipe(concat('public.js'))    //合并所有js到main.js
        .pipe(gulp.dest('dist/statics/js'))       //输出到文件夹
        .pipe(rename({suffix: '.min'}))   //rename压缩后的文件名
        .pipe(uglify())    //压缩
        .pipe(gulp.dest('dist/statics/js'))  //输出
});

//压缩，合并 config.js配置
gulp.task('convertConfigJs', function() {
    return gulp.src('config/*.js')      //需要操作的文件
        .pipe(concat('config.js'))    //合并所有js到main.js
        .pipe(gulp.dest('dist/config'))       //输出到文件夹
        .pipe(rename({suffix: '.min'}))   //rename压缩后的文件名
        .pipe(uglify())    //压缩
        .pipe(gulp.dest('dist/config'))  //输出
});

// 第三方框架移植
gulp.task('convertFramesJs',function() {
    return gulp.src(['src/frames/**/*'])// 该任务针对的文件
        .pipe(gulp.dest('dist/frames'))
});

// 组件样式
gulp.task('convertComponentsLess', function() {
    return gulp.src('src/components/**/*.less')//该任务针对的文件
        .pipe(plumber())
        .pipe(less())
        .pipe(minifycss())
        .pipe(gulp.dest('dist/components/'))
});
// 组件脚本
gulp.task('convertComponentsJs', function() {
    return gulp.src('src/components/**/*.js')//该任务针对的文件
        // .pipe(uglify())
        .pipe(gulp.dest('dist/components/'))
});
// 组件HTML
gulp.task('convertComponentsHtml', function() {
    return gulp.src('src/components/**/*.html')//该任务针对的文件
        .pipe(gulp.dest('dist/components/'))
});

// 生成文件hash字符串值、映射关系表
gulp.task('source-rev', function() {
    return gulp.src(['dist/**/**/*css', 'dist/**/**/*js'])
        .pipe(rev())
        .pipe(rev.manifest({
            path: 'rev-manifest.json'
        }))
        .pipe(gulp.dest('src/rev'));
});

// 根据映射关系表，将页面<link> <script>中url的文件名进行替换
gulp.task('source-rev-out', function() {
    return gulp.src(['src/rev/rev-manifest.json', 'src/view/**/*.html'])
        .pipe(revCollector())
        .pipe(gulp.dest('view'));
});

// 监视文件变化，自动执行任务
gulp.task('watch', function(){
    gulp.watch('src/less/*.less', ['convertLess']);
    gulp.watch('src/js/modules/*.js', ['convertPageJS']);
})

// 打包
gulp.task('build', ['convertPageJS', 'convertLess', 'convertComponentsLess', 'convertComponentsJs', 'convertComponentsHtml', 'convertPublicJs', 'convertConfigJs', 'convertFramesJs', 'source-rev', 'source-rev-out']);

// 开发
gulp.task('dev', ['convertPageJS', 'convertLess', 'watch']);
```