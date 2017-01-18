---
layout:     post
title:      "记一次rails 项目异常排查"
subtitle:   "你真的尊重最佳实践吗?"
header-img: img/pic/2015/10/kanasi2.jpg
tags: [ruby, rails, 调试]

---

## 背景

rails项目`data_service`, 在线上部署了2台机器, 两台都是rake的执行器, `LX-20-11` 和 `YJ-18-2`, 前者执行rake正常, 但是后者执行rake总是报错:

> $ bundle exec rake public:home_banner RAILS_ENV=production  
> rake aborted!  
> NoMethodError: undefined method `find_special_banner' for #<Class:0x007f1e1c96d728>  
> /tmp/cd-ruby-data_service/vendor/bundle/ruby/2.1.0/gems/activerecord-4.2.6/lib/active_record/dynamic_matchers.rb:26:in `method_missing'
> /tmp/cd-ruby-data_service/lib/data_service/public/home_banner.rb:11:in `block in index_background'
> ......

开发同学在线下一直没法重现, 因此怀疑是线上机器的问题, 请求运维协助排查机器差异, 但是线上机器系统, 环境几乎一模一样, 排查无果.

---

## 项目结构

`data_service` 是一个比较常规的rails 4 项目, 文件还是比较多, 我只列出涉及到的文件结构, 这是事后诸葛, 开始排查时并不清楚哪些文件是有关联的:

```
├── app
│   ├── models
│   │   ├── home_banner.rb
│   │   ├── st_superscript.rb
├── config
│   ├── initializers
│   │   ├── data_service_monitor_register.rb
├── lib
│   ├── data_service
│   │   ├── base.rb
│   │   ├── public
│   │   │   ├── home_banner.rb
│   │   │   ├── st_superscript.rb
│   ├── tasks
│   │   ├── public.rake
│   └── website
│       └── util
│           ├── image_uploadable.rb
```

略了很多不相关的文件.

---

## 排查过程

因为开发线下没有重现, 因此登陆线上堡垒机排查, 为了不影响现有服务, 我将项目复制了一份到/tmp下.

rake的入口在`lib/tasks/public.rake`, 代码很简单:

```ruby
# lib/tasks/public.rake
desc '首页品牌广告'
task :brand_banners => :environment do
  DataService::BrandBanners.save
end
```

从上面的错误栈中可以看到异常抛出点在`lib/data_service/public/home_banner.rb`, 涉及代码如下:

```ruby
# lib/data_service/public/home_banner.rb
class DataService::HomeBanner < DataService::Base
......
banner = HomeBanner.find_special_banner(banner_type, user_type).first
......
end
```

问题很明显, 类HomeBanner没有找到`find_special_banner`这个类方法, HomeBanner定义在`app/models/home_banner.rb`, 这个文件比较长, 有692行, 看了代码, 其中的确定义了类方法`find_special_banner`:

```ruby
# app/models/home_banner.rb
class HomeBanner < ActiveRecord::Base
  include Concerns::ImageUploadable
.......
  class << self
    def find_special_banner(banner_type = "", user_type = "")
      ......
    end
  end
.......
```

#### 类是否一致: `Object#object_id`

首先想到的是`lib/data_service/public/home_banner.rb` 中使用的HomeBanner和`app/models/home_banner.rb`里定义的HomeBanner不是同一个类, 因为[ruby常量查找](https://zhongfox.github.io/2013/03/21/ruby-constant-lookup/)有一个比较复杂的过程, 经常会出现查找不正确的情况. 

Ruby提供了`Object#object_id` 可以作为对象的标识, 因此先在以上2个文件中打印:

```ruby
# lib/data_service/public/home_banner.rb
class DataService::HomeBanner < DataService::Base
......
puts "1111111: #{HomeBanner.object_id}" # 调试代码
puts "2222222: #{::HomeBanner.object_id}" # 调试代码
banner = HomeBanner.find_special_banner(banner_type, user_type).first
......
end
```

```ruby
# app/models/home_banner.rb
class HomeBanner < ActiveRecord::Base
  puts "3333333: #{HomeBanner.object_id}" # 调试代码
  include Concerns::ImageUploadable
  .......
  class << self
    def find_special_banner(banner_type = "", user_type = "")
      ......
    end
  end
  .......
end
```


> 3333333: 70249085971460  
> 1111111: 70249085971460  
> 2222222: 70249085971460  
> rake aborted!  
> NoMethodError: undefined method `find_special_banner' for #<Class:0x007fc847a15808>
> ......

结果显示2个类的id一致, 排除常量查找问题.

#### 检视单键方法: `Object#singleton_methods(all=true) `

直接问题是找不到类方法, 那就到类方法的定义处去测试一下, ruby的类方法就是类的单键方法, 可以使用`Object#singleton_methods(all=true)`进行检视:

```ruby
# app/models/home_banner.rb
class HomeBanner < ActiveRecord::Base
puts "3333333: #{HomeBanner.object_id}" # 调试代码
  include Concerns::ImageUploadable
.......
  class << self
    def find_special_banner(banner_type = "", user_type = "")
      ......
    end
    puts "all HomeBanner class methods: #{HomeBanner.singleton_methods(false)}" # 调试代码
  end
.......
```

> 3333333: 69955224157780  
> rake aborted!  
> NoMethodError: undefined method `find_special_banner' for #<Class:0x007f3f70927ca8>
> ......

不过非常意外的是, 最下面的调试代码在rake报错前没有任何输出, 但是上面一行调试代码却是有输出的. 排查了一会, 基本可以确定, rake 在加载HomeBanner这个类的中途发生了异常(在定义类方法前), 但是这个异常被吃掉了, 导致HomeBanner加载发生异常没有抛出. 最终抛出的却是使用HomeBanner时缺乏类方法的错误.

#### 确定代码调用栈 `Kernel#caller`

异常被rescue但是没有正确处理或者上报, 是导致调试困难的常见原因, 麻烦的是根本不知道异常是在哪里吃掉的, 也就不知道在加载HomeBanner时发生了什么异常.

还好强大的Ruby提供了`Kernel#caller`, 可以输出代码的调用栈. 从上面的排查可以看出, HomeBanner 的确是加载并执行了, 只是在中途停止了, 没有加载完, 因此可以在文件头打印出调用栈, 看看是哪行该死的代码吃掉了异常:


```ruby
# app/models/home_banner.rb
puts caller # 调试代码
class HomeBanner < ActiveRecord::Base
  include Concerns::ImageUploadable
  .......
  class << self
    def find_special_banner(banner_type = "", user_type = "")
      ......
    end
  end
  .......
end
```

输出结果:

```
vendor/bundle/ruby/2.1.0/gems/activesupport-4.2.6/lib/active_support/dependencies.rb:274:in 'require'
vendor/bundle/ruby/2.1.0/gems/activesupport-4.2.6/lib/active_support/dependencies.rb:274:in 'block in require'
vendor/bundle/ruby/2.1.0/gems/activesupport-4.2.6/lib/active_support/dependencies.rb:240:in 'load_dependency'
vendor/bundle/ruby/2.1.0/gems/activesupport-4.2.6/lib/active_support/dependencies.rb:274:in 'require'
vendor/bundle/ruby/2.1.0/gems/activesupport-4.2.6/lib/active_support/dependencies.rb:360:in 'require_or_load'
vendor/bundle/ruby/2.1.0/gems/activesupport-4.2.6/lib/active_support/dependencies.rb:494:in 'load_missing_constant'
vendor/bundle/ruby/2.1.0/gems/activesupport-4.2.6/lib/active_support/dependencies.rb:184:in 'const_missing'
lib/data_service/base.rb:81:in 'const_get'
lib/data_service/base.rb:81:in 'inherited'
lib/data_service/public/home_banner.rb:1:in '<top (required)>'
```

可以看到项目加载HomeBanner的入口在`lib/data_service/base.rb`:

```ruby
# lib/data_service/base.rb
name = subclass.name.demodulize
top_model = Object.const_get(name) rescue nil
```

这里的确有`rescue`!, 这个需求是当获取不到name对应的Activerecord 模型, 就直接返回nil. 只是这里结果并不是找不到HomeBanner, 而是找到HomeBanner后, 加载时, HmeBanner 里有异常.

#### 检视异常堆栈: `Exception#backtrace`

rake加载HomeBanner会出错, 但是这个异常在rails server或者rails console里并没有重现, 因此直接的办法是在异常点检视异常信息, Ruby异常类Exception提供了丰富的检视方法, 简化的调试代码如下:

```ruby
# lib/data_service/base.rb
name = subclass.name.demodulize
begin
  top_model = Object.const_get(name)
rescue => e
  puts e.message
  puts e.backtrace
end
```

输出结果:

```
uninitialized constant Concerns::ImageUploadable
app/models/home_banner.rb:4:in `<class:HomeBanner>'
app/models/home_banner.rb:3:in `<top (required)>'
lib/data_service/base.rb:84:in `const_get'
lib/data_service/base.rb:84:in `inherited'
lib/data_service/public/home_banner.rb:1:in `<top (required)>'
config/initializers/data_service_monitor_register.rb:4:in `block (2 levels) in <top (required)>'
```

最终发现是`app/models/home_banner.rb` 里`include Concerns::ImageUploadable` 无法找到这个模块!

在`app/models/concerns/` 下找了一下, 并没发现这个模块, 最后通过grep发现这个文件躺在`lib/website/util/image_uploadable.rb`, 这是违反「约定大于配置」的后果.

#### 猜测与验证

机器`YJ-18-2`的`app/models/home_banner.rb`无法找到`lib/website/util/image_uploadable.rb`, 但是机器 `LX-20-11`是正常的, 因此到正常的机器`LX-20-11`中, 再次使用`caller`定位到`lib/website/util/image_uploadable.rb` 在另一个model `app/models/st_superscript.rb` 中require了;

```ruby
# app/models/st_superscript.rb
require "#{Rails.root}/lib/website/util/image_uploadable.rb"
class StSuperscript < ActiveRecord::Base
```

第一感觉是这里的require不同寻常, `image_uploadable.rb`是上传文件的通用concern, 但是单单在其中一个使用model`app/models/st_superscript.rb`中require, `app/models/home_banner.rb` 中并没有require.

到这里基本上可以猜测到, 正常机器`LX-20-11` 是先加载了`app/models/st_superscript.rb` 然后再加载了`app/models/home_banner.rb`, 依赖的concern是在前者中进行require, 而出错的机器`YJ-18-2`的加载顺序刚好相反, 因此导致`app/models/home_banner.rb`无法找到依赖concern.

在两台机器的`st_superscript.rb`和`home_banner.rb`分别增加调试代码, 两台机器打印结果的顺序不同, 证明了之前的猜测.

#### 文件加载顺序

相同的代码, 在几乎相同的机器上, 文件加载顺序不一致, 这个比较罕见, 不过可能的原因也有很多, 很多shell 命令都不保证文件遍历的顺序.

通过`caller` 继续确定2个model的加载入口, 在文件`config/initializers/data_service_monitor_register.rb`里:

```ruby
# config/initializers/data_service_monitor_register.rb
DataService::Application.config.to_prepare do
  ......
  Dir[File.join(Rails.root, 'lib/data_service/public/*.rb')].each { |file| load file }
  ......
end
```

`lib/data_service/public` 下的`home_banner.rb` 和`st_superscript.rb`  会分别加载`app/models`下的同名文件.

看到`Dir`, 猜测很大的可能是这个api返回的文件列表顺序不一致, 查看api文档:

```
Dir[ string [, string ...] ] → array
Equivalent to calling Dir.glob([string,...],0).

......

glob( pattern, [flags] ) → matches
glob( pattern, [flags] ) { |filename| block } → nil
Expands pattern, which is an Array of patterns or a pattern String, and returns the results as matches or as arguments given to the block.

Note that this pattern is not a regexp, it’s closer to a shell glob. See File.fnmatch for the meaning of the flags parameter. Note that case sensitivity depends on your system (so File::FNM_CASEFOLD is ignored), as does the order in which the results are returned.
```

可以看到的是, glob 并没有保证返回的顺序, 返回顺序和机器, 系统相关!

还是不放心? 在两台机器的irb分别执行一下:

出错的机器:

```
> Dir['lib/data_service/public/*.rb'].grep /st_superscript|home_banner/
=> [
  "lib/data_service/public/home_banner.rb",
   "lib/data_service/public/st_superscript.rb"
]
```

正确机器:

```
> Dir['lib/data_service/public/*.rb'].grep /st_superscript|home_banner/
=> [
   "lib/data_service/public/st_superscript.rb",
  "lib/data_service/public/home_banner.rb"
]
```

顺序的确不同!

---

## 修复

通过排查过程, 可以发现, 代码至少违反了以下最佳实践:

* 吃掉异常, 屏蔽掉了底层异常的现象, 导致出错后排查困难.

* 违反了「约定大于配置」, Concern模块`image_uploadable.rb` 按照约定应该放到`app/models/concerns/`, 这个目录属于`eager_load_paths`, 目录内的模块会被自动require,  但是项目却把`image_uploadable.rb`放到了`lib/website/util/image_uploadable.rb`, 因此需要在使用的地方去手动require, 但是没处理好加载顺序. 导致异常.

* 项目在开发测试阶段间接地依赖了`Dir`返回的文件顺序, 虽然项目本意并不关心`lib/data_service/public/`下的文件加载顺序, 程序在线下开发和测试测试阶段能正常运行完全是一个「巧合」. 如果依赖文件加载顺序, 应该自己在结果后面进行sort.

解决法办法很简单: 把`image_uploadable.rb` 移到它应该在的位置`app/models/concerns/image_uploadable.rb`, 问题就解决了.

---

## 总结

1. 你真的理解「约定大于配置」的原因吗? 你真的尊重「约定大于配置」吗?


   违反rails的「约定」会带来沉重的代价, concern不在它该在的位置, 没法自动加载, 需要在model写奇怪的代码进行手动require.

   这个问题发生在一两个月前, 导致若干rake异常, 只有部分机器报错. 开发排查, 运维排查, 开发运维扯皮, 最后迫不得已上线调试, 前后也花了将近2个小时.

2. 开发应该对「奇怪」的代码有反感.

   我觉得这里项目里最奇怪的代码是在`app/models/st_superscript.rb`里, 手动require了大部分model都要使用公共Concern `require "lib/website/util/image_uploadable.rb"`, 这应该是在开发途中, 违反「约定大于配置」的一个补丁, 开发应该对这种奇怪的代码多去刨根问底.

3. 对涉及IO遍历的返回值, 需要考虑一下顺序是否会影响最终结果, 如果api文档没有保证顺序, 那么就是不确定的, 自己加上`sort`是个不赖的办法. 不要让你的代码依赖「巧合」运行.

4. Ruby 强大元编程非常利于调试, 要善加利用.

   调试前后用到的主要技巧:

   * 判断对象一致性: `Object#object_id`
   * 检视单键方法: `Object#singleton_methods`
   * 确定代码调用栈: `Kernel#caller`
   * 检视异常堆栈: `Exception#backtrace`

5. 调试还是需要一些合理猜测和验证.

6. 尊重结果, 按图索骥.

   以后遇到类似bug, 不要轻易说「你的环境有问题吧?」 「在我这里可是好的」
