---
layout:       post
title:        "Windows 安装 Jekyll"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - jekyll
---

### 1. 安装 Ruby

* 下载安装 RubyInstallers ，推荐下载 Ruby+Devkit  
  <https://rubyinstaller.org/downloads/>  
  ![](/img/jekyll/jekyll01.png)  

* 安装：使用默认路径即可，避免出错；勾选添加到PATH，就不用手动添加环境变量了
  ![](/img/jekyll/jekyll02.png)  


* 这里需要勾选安装msys2，后面安装gem和jekyll时会用到：
  ![](/img/jekyll/jekyll03.png)  
  ![](/img/jekyll/jekyll04.png)  
  ![](/img/jekyll/jekyll05.png)  

### 2. 安装 RubyGems
<https://rubygems.org/pages/download>  

* 下载后解压到任意路径；进入解压目录，打开cmd，输入以下命令：
  ```bash
  > ruby setup.rb
  ```

* 改为国内源

  ```bash
  > gem sources -l
  *** CURRENT SOURCES ***
  
  https://rubygems.org/
  
  > gem sources -a https://gems.ruby-china.com/
  https://gems.ruby-china.com/ added to sources
  
  > gem sources --remove https://rubygems.org/
  https://rubygems.org/ removed from sources
  
  > gem sources -l
  *** CURRENT SOURCES ***
  
  https://gems.ruby-china.com/
  
  > bundle config mirror.https://rubygems.org https://gems.ruby-china.com
  ```



### 3. 安装 Jekyll

* 在 cmd 中输入：
  ```bash
  > gem install jekyll
  
  > jekyll -v
  jekyll 4.3.3
  ```

### 4. 安装 jekyll-paginate
  ```bash
  > gem install jekyll-paginate
  ```

### 5. 安装 bundler
  ```bash
  > gem install bundler
  
  > bundler -v
  Bundler version 2.5.18
  ```

### 6. 新建 jekyll 模板

* 安装完成，我们可以用 jekyll 命令创建一个博客模板，进入一个目录，打开命令行执行：

  ```
  > jekyll new testblog
  > cd testblog
  > jekyll serve
  ```

* 在浏览器输入 http://127.0.0.1:4000/ 即可浏览刚刚创建的模板

### 7. 遇到的问题

* make: *** [Makefile:248：rb_monitor.o] 错误 1

  ```bash
  C:\Users\vito\Downloads\tmp> jekyll new testblog
  Running bundle install in C:/Users/vito/Downloads/tmp/testblog...
    Bundler: Fetching gem metadata from https://rubygems.org/............
    Bundler: Resolving dependencies...
    Bundler: Installing wdm 0.1.1 with native extensionsGem::Ext::BuildError: ERROR: Failed to build gem native extension.
    Bundler:
    Bundler: current directory: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/wdm-0.1.1/ext/wdm
    Bundler: C:/Ruby33-x64/bin/ruby.exe extconf.rb
    Bundler: checking for -lkernel32... yes
    Bundler: checking for windows.h... yes
    Bundler: checking for ruby.h... yes
    Bundler: checking for HAVE_RUBY_ENCODING_H... yes
    Bundler: checking for rb_thread_call_without_gvl()... yes
    Bundler: creating Makefile
    Bundler:
    Bundler: current directory: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/wdm-0.1.1/ext/wdm
    Bundler: make DESTDIR\= sitearchdir\=./.gem.20240901-10460-wtt61u
    Bundler: sitelibdir\=./.gem.20240901-10460-wtt61u clean
    Bundler:
    Bundler: current directory: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/wdm-0.1.1/ext/wdm
    Bundler: make DESTDIR\= sitearchdir\=./.gem.20240901-10460-wtt61u
    Bundler: sitelibdir\=./.gem.20240901-10460-wtt61u
    Bundler: generating wdm_ext-x64-mingw-ucrt.def
    Bundler: compiling entry.c
    Bundler: compiling memory.c
    Bundler: compiling monitor.c
    Bundler: compiling queue.c
    Bundler: compiling rb_change.c
    Bundler: rb_change.c: In function 'extract_absolute_path_from_notification':
    Bundler: rb_change.c:139:5: warning: 'RB_OBJ_TAINT' is deprecated: taintedness turned out
    Bundler: to be a wrong idea. [-Wdeprecated-declarations]
    Bundler: 139 | OBJ_TAINT(path);
    Bundler: | ^~~~~~~~~
    Bundler: In file included from
    Bundler: C:/Ruby33-x64/include/ruby-3.3.0/ruby/internal/core/rstring.h:30,
    Bundler: from
    Bundler: C:/Ruby33-x64/include/ruby-3.3.0/ruby/internal/arithmetic/char.h:29,
    Bundler: from
    Bundler: C:/Ruby33-x64/include/ruby-3.3.0/ruby/internal/arithmetic.h:24,
    Bundler: from C:/Ruby33-x64/include/ruby-3.3.0/ruby/ruby.h:28,
    Bundler: from C:/Ruby33-x64/include/ruby-3.3.0/ruby.h:38,
    Bundler: from wdm.h:22,
    Bundler: from rb_change.c:4:
    Bundler: C:/Ruby33-x64/include/ruby-3.3.0/ruby/internal/fl_type.h:824:1: note: declared
    Bundler: here
    Bundler: 824 | RB_OBJ_TAINT(VALUE obj)
    Bundler: | ^~~~~~~~~~~~
    Bundler: compiling rb_monitor.c
    Bundler: rb_monitor.c: In function 'rb_monitor_run_bang':
    Bundler: rb_monitor.c:509:29: error: implicit declaration of function
    Bundler: 'rb_thread_call_without_gvl' [-Wimplicit-function-declaration]
    Bundler: 509 | waiting_succeeded = rb_thread_call_without_gvl(wait_for_changes,
    Bundler: monitor->process_event, stop_monitoring, monitor);
    Bundler: | ^~~~~~~~~~~~~~~~~~~~~~~~~~
    Bundler: make: *** [Makefile:248：rb_monitor.o] 错误 1
    Bundler:
    Bundler: make failed, exit code 2
    Bundler:
    Bundler: Gem files will remain installed in
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/wdm-0.1.1 for inspection.
    Bundler: Results logged to
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/extensions/x64-mingw-ucrt/3.3.0/wdm-0.1.1/gem_make.out
    Bundler:
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:125:in `run'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:51:in `block in
    Bundler: make'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:43:in `each'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:43:in `make'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/ext_conf_builder.rb:42:in
    Bundler: `build'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:193:in
    Bundler: `build_extension'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:227:in `block
    Bundler: in build_extensions'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:224:in `each'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/ext/builder.rb:224:in
    Bundler: `build_extensions'
    Bundler: C:/Ruby33-x64/lib/ruby/site_ruby/3.3.0/rubygems/installer.rb:853:in
    Bundler: `build_extensions'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/rubygems_gem_installer.rb:109:in
    Bundler: `build_extensions'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/rubygems_gem_installer.rb:28:in
    Bundler: `install'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/source/rubygems.rb:205:in
    Bundler: `install'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/installer/gem_installer.rb:54:in
    Bundler: `install'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/installer/gem_installer.rb:16:in
    Bundler: `install_from_spec'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/installer/parallel_installer.rb:132:in
    Bundler: `do_install'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/installer/parallel_installer.rb:123:in
    Bundler: `block in worker_pool'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/worker.rb:62:in
    Bundler: `apply_func'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/worker.rb:57:in
    Bundler: `block in process_queue'
    Bundler: <internal:kernel>:187:in `loop'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/worker.rb:54:in
    Bundler: `process_queue'
    Bundler: C:/Ruby33-x64/lib/ruby/gems/3.3.0/gems/bundler-2.5.18/lib/bundler/worker.rb:90:in
    Bundler: `block (2 levels) in create_threads'
    Bundler:
    Bundler: An error occurred while installing wdm (0.1.1), and Bundler cannot continue.
    Bundler:
    Bundler: In Gemfile:
    Bundler: wdm
  ```

  解决办法：

  ```bash
  > gem install wdm
  Fetching wdm-0.2.0.gem
  Temporarily enhancing PATH for MSYS/MINGW...
  Building native extensions. This could take a while...
  Successfully installed wdm-0.2.0
  Parsing documentation for wdm-0.2.0
  Installing ri documentation for wdm-0.2.0
  Done installing documentation for wdm after 0 seconds
  1 gem installed
  
  > cd testblog
  
  > notepad Gemfile
  将：
  gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
  改为：
  gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]
  
  > bundle exec jekyll serve
  ```

* Deprecation Warning: Using / for division outside of calc() is deprecated and will be removed in Dart Sass 2.0.0.

  ```bash
  > notepad _config.yml
  
  sass:
    quiet_deps: true
  ```

