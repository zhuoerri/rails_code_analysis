# 环境
* ruby 2.3.1
* rails 5.2.0

# 简介
ruby本身加载第三方库的机制很简单。就是$LOAD_PATH

但为了更便捷，无冲突和高性能，开发者们陆续开发了 rubygems, bundle, 再到今天要讲的bootsnap。

# 预备知识
## $LOAD_PATH

```ruby
  # 假设当前文件夹/test, 下有2个文件 
  # a.rb ，内容为 puts '找到a了'
  # b.rb ，内容为 puts '找到b了'
  $ irb
  > require '/test/b'
  # 找到b了
  
  require 'a'
  # LoadError: cannot load such file -- a
  
  # 加载b时，因为是绝对路径，直接就找到b.rb文件了
  # 加载a时，ruby找不到这个文件而报错
  
  # ruby不知道去哪个文件夹下找 a.rb ？这就是$LOAD_PATH的作用。
  # irb内输入$LOAD_PATH 
  $LOAD_PATH
  # 返回 ["/Users/qinyang/.rvm/gems/ruby-2.3.1@global/gems/did_you_mean-1.0.0/lib", "/Users/qinyang/.rvm/rubies/ruby-2.3.1/lib/ruby/site_ruby/2.3.0", "/Users/qinyang/.rvm/rubies/ruby-2.3.1/lib/ruby/site_ruby/2.3.0/x86_64-darwin18"]
  # 可以看到$LOAD_PATH是个数组，存储了很多文件夹路径。
  # ruby会依次去每个文件夹下，尝试寻找 a.rb
  
  # 把当前文件夹加入$LOAD_PATH, 这样就能找到当前文件夹下的 a.rb
  $LOAD_PATH << '/test'
  
  # 再次 require，就不报错了
  require 'a'
  # 找到a了
```

## rubygems
为了方便分享自己写的库给别人使用，开发者开发了[rubygems](https://github.com/rubygems/rubygems), 也就是平时`gem install xxxxx`的`gem`这个命令

gem会去[rubygems.org](https://rubygems.org/)下载第三方库到本地。

虽然rubygems.org需要翻墙，但还好网站代码是开源的 [https://github.com/rubygems/rubygems.org](https://github.com/rubygems/rubygems.org),可搭建自己的rubygems.org，就像ruby-china搭建的一样[http://gems.ruby-china.com/](http://gems.ruby-china.com/)

```ruby
# 除了提供gem这个命令便于我们下载第三方库，rubygems还覆盖了原本的 require 方法

# 先下载pry
$ gem install pry
# 安装到了我本地的 /Users/qinyang/.rvm/gems/ruby-2.3.1/gems/pry-0.12.2/lib

$ irb
> $LOAD_PATH
# 返回的数组里没有 /Users/qinyang/.rvm/gems/ruby-2.3.1/gems/pry-0.12.2/lib
> require 'pry'
# 却返回 true，加载成功
> $LOAD_PATH
# 再查看$LOAD_PATH时，/Users/qinyang/.rvm/gems/ruby-2.3.1/gems/pry-0.12.2/lib 却存在于返回的数组中了

# 用method.source_location查看方法定义所在文件，及文件第几行
> method(:require).source_location
=> ["/Users/qinyang/.rvm/rubies/ruby-2.3.1/lib/ruby/site_ruby/2.3.0/rubygems/core_ext/kernel_require.rb", 39]
```
```ruby
# https://github.com/rubygems/rubygems/blob/2.7/lib/rubygems/core_ext/kernel_require.rb
module Kernel

  # 把ruby原本的require方法取别名为gem_original_require
  unless defined?(gem_original_require)
    alias gem_original_require require
    private :gem_original_require
  end
  
  # 覆盖require方法
  def require path
    # 省略。。。。。
    # ........
    
  # 如果原生require方法在$LOAD_PATH中没有找到 path, 就rescue 
  rescue LoadError => load_error
    RUBYGEMS_ACTIVATION_MONITOR.enter

    begin
      if load_error.message.start_with?("Could not find") or
          
          # Gem.try_activate(path) 尝试从已下载的库里去找path。若找到，则把文件夹加入$LOAD_PATH
          (load_error.message.end_with?(path) and Gem.try_activate(path)) then
        require_again = true
      end
    ensure
      RUBYGEMS_ACTIVATION_MONITOR.exit
    end

    # 若在下载的库里找到path，则再执行一次require
    return gem_original_require(path) if require_again

    raise load_error
  end

end
```

## bundler
bundler会在Gemfile里所列举的gem中，找到能相互不冲突的版本，并下载安装。
但本文重点是介绍`$LOAD_PATH`。所以只简单介绍bundler对`$LOAD_PATH`的操作。

```ruby
# 找一个rails项目，打开config/boot.rb.下面是一个rails4的项目的config/boot.rb

# Set up gems listed in the Gemfile.
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)

# 添加puts语句，记录执行bundler/setup前$LOAD_PATH有多少个
puts 'before bundler/setup $LOAD_PATH is ' + $LOAD_PATH.count.to_s
puts '=' * 100

require 'bundler/setup' if File.exist?(ENV['BUNDLE_GEMFILE'])

# 添加puts语句，记录执行bundler/setup后$LOAD_PATH有多少个
puts 'after bundler/setup $LOAD_PATH is ' + $LOAD_PATH.count.to_s
```

```ruby
# 在终端执行rails c之类的操作，注意用bin/rails c,而不是rails c。这两者执行顺序上有差异
$ bin/rails c
# before bundler/setup $LOAD_PATH is 9
# ================================================================
# after bundler/setup $LOAD_PATH is 258

# 可以看出，bundler不止下载gem，还会在项目启动时，把gem的路径加入$LOAD_PATH中,方便之后的require.
# config/application.rb中Bundler.require(*Rails.groups) 这句，就会去require所有列在gemfile里面的gem
```

# bootsnap
当$LOAD_PATH存有越来越多的文件夹时，每次require在最坏的情况下就需要把所有文件夹都查找一遍。假设$LOAD\_PATH里有100个文件夹，并且需要require 100个库，则时间复杂度就是100x100,即n(100)的平方。

为了require时提高查找文件的性能，rails5添加了bootsnap组件。工作原理是提前把$LOAD\_PATH下的所有文件夹都扫描一遍，记录下每个文件夹下都有哪些文件，记为hash，序列化存入 `项目/tmp/cache/bootsnap-load-path-cache`内。再反过来，交换hash的key，value，记录每个文件所属的文件夹，存入一个新的hash内。

每次require文件时，只需去新hash内找出对应文件夹，即可拼凑出完整文件路径。改为直接require这个文件路径，无需再遍历$LOAD\_PATH

```ruby
# 进入rails c
> MessagePack.load(File.binread('./tmp/cache/bootsnap-load-path-cache'))
# 返回一个hash，key是文件夹路径，value是文件夹内的所有文件。

> Bootsnap::LoadPathCache.load_path_cache.instance_eval do @index end
# 返回一个hash，key是文件名，value是所属的文件夹
```
### bootsnap源码
```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/setup.rb
# 初始化配置 cache_dir 是 tmp/cache 文件夹 
Bootsnap.setup(
  cache_dir:            cache_dir,
  development_mode:     development_mode,
  load_path_cache:      true,
  autoload_paths_cache: true, # assume rails. open to PRs to impl. detection
  disable_trace:        false,
  compile_cache_iseq:   iseq_cache_enabled,
  compile_cache_yaml:   true,
)
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap.rb
module Bootsnap
  def self.setup(
    cache_dir:,
    development_mode: true,
    load_path_cache: true,
    autoload_paths_cache: true,
    disable_trace: false,
    compile_cache_iseq: true,
    compile_cache_yaml: true
  )
    if autoload_paths_cache && !load_path_cache
      raise(InvalidConfiguration, "feature 'autoload_paths_cache' depends on feature 'load_path_cache'")
    end

    setup_disable_trace if disable_trace

    # 再调用loadPathCache#setup
    Bootsnap::LoadPathCache.setup(
      cache_path:       cache_dir + '/bootsnap-load-path-cache',
      development_mode: development_mode,
      active_support:   autoload_paths_cache
    ) if load_path_cache
    
    # 省略。。。。。。
  end
end
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/load_path_cache.rb
module Bootsnap
  module LoadPathCache
    class << self
      def setup(cache_path:, development_mode:, active_support: true)
        unless supported?
          warn("[bootsnap/setup] Load path caching is not supported on this implementation of Ruby") if $VERBOSE
          return
        end

        # Store负责将hash序列化和反序列化，存入或读取tmp/cache/bootsnap-load-path-cache
        # Store读取和写入的方法见下面
        store = Store.new(cache_path)

        # 记录哪些gem已经被require了,防止重复require，具体用法见下面覆盖require方法时
        @loaded_features_index = LoadedFeaturesIndex.new
        
        @realpath_cache = RealpathCache.new

        # Cache负责生成hash，具体生成方法见下面
        @load_path_cache = Cache.new(store, $LOAD_PATH, development_mode: development_mode)
        
        require_relative('load_path_cache/core_ext/kernel_require')
        require_relative('load_path_cache/core_ext/loaded_features')
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/load_path_cache/store.rb
module Bootsnap
  module LoadPathCache
    class Store
    
      def load_data
        # 用MessagePack把tmp下的文件反序列化成hash读取
        @data = begin
          MessagePack.load(File.binread(@store_path))
          # handle malformed data due to upgrade incompatability
        rescue Errno::ENOENT, MessagePack::MalformedFormatError, MessagePack::UnknownExtTypeError, EOFError
          {}
        rescue ArgumentError => e
          e.message =~ /negative array size/ ? {} : raise
        end
      end
      
      def dump_data
        # Change contents atomically so other processes can't get invalid
        # caches if they read at an inopportune time.
        tmp = "#{@store_path}.#{Process.pid}.#{(rand * 100000).to_i}.tmp"
        FileUtils.mkpath(File.dirname(tmp))
        exclusive_write = File::Constants::CREAT | File::Constants::EXCL | File::Constants::WRONLY
        # `encoding:` looks redundant wrt `binwrite`, but necessary on windows
        # because binary is part of mode.
        
        # 用MessagePack把hash序列化成文件
        File.binwrite(tmp, MessagePack.dump(@data), mode: exclusive_write, encoding: Encoding::BINARY)
        FileUtils.mv(tmp, @store_path)
      rescue Errno::EEXIST
        retry
      end
    end
  end
end
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/load_path_cache/cache.rb
module Bootsnap
  module LoadPathCache
    class Cache
      def initialize(store, path_obj, development_mode: false)
        @development_mode = development_mode
        @store = store
        @mutex = defined?(::Mutex) ? ::Mutex.new : ::Thread::Mutex.new # TODO: Remove once Ruby 2.2 support is dropped.
        @path_obj = path_obj.map! { |f| File.exist?(f) ? File.realpath(f) : f }
        @has_relative_paths = nil
        reinitialize
      end
      
      # 扫描@path_obj，也就是$LOAD_PATH,生成hash
      def reinitialize(path_obj = @path_obj)
        @mutex.synchronize do
          @path_obj = path_obj
          
          # 注册事件回调函数，当$LOAD_PATH有修改时,重新reinitialize
          ChangeObserver.register(self, @path_obj)
          
          @index = {}
          @dirs = {}
          @generated_at = now
          
          # 再调puash_paths_locked，负责实际扫描$LOAD_PATH
          push_paths_locked(*@path_obj)
        end
      end
      
      def push_paths_locked(*paths)
        @store.transaction do
          paths.map(&:to_s).each do |path|
          
            # Path是对文件夹路径的一个封装类
            p = Path.new(path)
            
            @has_relative_paths = true if p.relative?
            next if p.non_directory?
            expanded_path = p.expanded_path
            
            # 扫描p下所有文件和文件夹，存入@store，并返回。
            # entries代表p下的文件，dirs代表p下的文件夹
            entries, dirs = p.entries_and_dirs(@store)
            
            dirs.each    { |dir| @dirs[dir]  ||= path }
            
            # 生成前文提到的新hash，也就是@index。key是p下的文件，value是p的绝对路径，
            entries.each { |rel| @index[rel] ||= expanded_path }
          end
        end
      end
    end
  end
end
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/load_path_cache/path.rb
module Bootsnap
  module LoadPathCache
    class Path
      def entries_and_dirs(store)
        # 省略。。。。
        
        # 调用scan！，扫描文件夹下文件
        entries, dirs = scan!
        
        # 把扫描结果序列化存入store
        store.set(expanded_path, [current_mtime, entries, dirs])
        
        [entries, dirs]
      end
      
      private
      def scan!
      
        # 再调call方法, 扫描文件夹下文件
        PathScanner.call(expanded_path)
      end
    end
  end
end
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/load_path_cache/path_scanner.rb
module Bootsnap
  module LoadPathCache
    module PathScanner
      ALL_FILES = "/{,**/*/**/}*"
      REQUIRABLE_EXTENSIONS = [DOT_RB] + DL_EXTENSIONS
      
      def self.call(path)
        # 省略。。。。
        
        # 遍历path下的文件
        Dir.glob(path + ALL_FILES).each do |absolute_path|
          
          # 如果文件是属于bundle这个gem的，就忽略
          next if contains_bundle_path && absolute_path.start_with?(BUNDLE_PATH)
          
          relative_path = absolute_path.slice(relative_slice)

          if File.directory?(absolute_path)
            dirs << relative_path
            
          # 如果文件的后缀是 ".rb", ".bundle" 就存入返回结果
          # 因为我是mac电脑 ".bundle"是mac系统的库文件格式
          # linux系统的话，应该是 ".so"
          elsif REQUIRABLE_EXTENSIONS.include?(File.extname(relative_path))
            requirables << relative_path
          end
        end

        [requirables, dirs]
      end
    end
  end
end
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/load_path_cache/core_ext/kernel_require.rb
module Kernel

  # ruby原生require取了个别名
  alias_method(:require_without_bootsnap, :require)

  def require(path)
    # 如果已经require过了，则忽略
    return false if Bootsnap::LoadPathCache.loaded_features_index.key?(path)

    # 覆盖ruby的原生require方法，改为调用find方法。
    # 即从@index 这个hash中查找文件对应的绝对路径
    if (resolved = Bootsnap::LoadPathCache.load_path_cache.find(path))
      
      # 如果找到了，再调require_with_bootsnap进行require
      return require_with_bootsnap_lfi(path, resolved)
    end

    # 省略。。。。
  end
  
  def require_with_bootsnap_lfi(path, resolved = nil)
  
    # 把path加入loaded_features_index，记为已加载
    Bootsnap::LoadPathCache.loaded_features_index.register(path, resolved) do
      
      # ruby原生require方法的别名
      require_without_bootsnap(resolved || path)
    end
  end
end
```

```ruby
# https://github.com/Shopify/bootsnap/blob/v1.4.4/lib/bootsnap/load_path_cache/cache.rb
module Bootsnap
  module LoadPathCache
    class Cache
    
      def find(feature)
        # 省略。。。。
        
        feature = feature.to_s
        return feature if absolute_path?(feature)
        return expand_path(feature) if feature.start_with?('./')
        @mutex.synchronize do
        
          # 再调search_index
          x = search_index(feature)
          return x if x
          
          # 省略。。。。。
        end
        
        # 省略。。。。
      end
      
      def search_index(f)
      
        # 加各种可能的后缀，调try_index
        try_index(f + DOT_RB) || try_index(f + DLEXT) || try_index(f + DLEXT2) || try_index(f)
      end
      
      def try_index(f)
      
        # 从@index从获取gem所在文件夹的绝对路径，并凭凑出文件的绝对路径
        if (p = @index[f])
          p + '/' + f
        end
      end
    end
  end
end
```

