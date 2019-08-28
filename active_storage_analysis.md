# 环境
* ruby 2.3.1
* rails 5.2.0


# 简介
ActiveStorage 是 rails 官方的附件保存组件。可替代 carrierwave 等gem。

# 配置
* 执行迁移

```ruby
rake active_storage:install
rake db:migrate
```

* `config/storage.yml`中申明有哪些附件保存方式，例如本地文件系统，amazon云存储等。

```ruby
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
 
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>
 
amazon:
  service: S3
  access_key_id: ""
  secret_access_key: ""
  bucket: ""
  region: "" # e.g. 'us-east-1'
```

* `config/environments/development.rb`中配置使用什么保存方式

```ruby
# Store files locally.
config.active_storage.service = :local
```

* 在model中使用 `has_one_attached ` ,或 `has_many_attached `

```ruby
class Post < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :addresses
end
```

# 使用
```ruby
############ 上传文件
# controller里如往常一样create
Post.create(params.permit(:avatar))

# 或者rails c里直接用attach方法
post.avatar.attach(params[:avatar])
post.avatar.attach(io: File.open("/path/to/file.jpg"), filename: "file.jpg", content_type: "image/jpg")


############ 下载文件
# 直接获取二进制文件内容
binary = post.avatar.download

# 或用 rails_blob_path 生成下载地址
# 这个下载地址会重定向到一个加密地址，加密地址有时效，默认5分钟
rails_blob_path(post.avatar, disposition: "attachment")

```

# 数据库
migration生成了两张表:

* `active_storage_attachments` 用来跟其他model做多态关联
  * 用 record\_type, record\_id [多态关联](https://ruby-china.github.io/rails-guides/association_basics.html#polymorphic-associations) 别的表
  * 用 blod_id, 关联 `active_storage_blobs` 表
* `active_storage_blobs` 代表文件
  * key字段：文件的唯一标识，可以理解为文件的id，用于下载或更新文件时根据key查找文件
  * checksum字段： 校验文件内容完整性的
  * 其他的文件名，文件大小，文件类型等字段

# 源码
只讲最基础的上传，下载的源码。其他如生成缩略图，图片切割成不同尺寸等功能，忽略。
## has\_one\_attached 等宏方法

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/lib/active_storage/attached/macros.rb
module ActiveStorage
  module Attached::Macros
    def has_one_attached(name, dependent: :purge_later)
      # 生成对应getter，setter方法
      generated_association_methods.class_eval <<-CODE, __FILE__, __LINE__ + 1
        def #{name}
          @active_storage_attached_#{name} ||= ActiveStorage::Attached::One.new("#{name}", self, dependent: #{dependent == :purge_later ? ":purge_later" : "false"})
        end
        def #{name}=(attachable)
          #{name}.attach(attachable)
        end
      CODE

      # 跟attachments, blobs表关联
      has_one :"#{name}_attachment", -> { where(name: name) }, class_name: "ActiveStorage::Attachment", as: :record, inverse_of: :record, dependent: false
      has_one :"#{name}_blob", through: :"#{name}_attachment", class_name: "ActiveStorage::Blob", source: :blob

      # 增加一个scope，用来防止N+1查询
      scope :"with_attached_#{name}", -> { includes("#{name}_attachment": :blob) }

      # 根据选项，model被删除后，是只解除数据库关联，还是把文件也删除
      if dependent == :purge_later
        after_destroy_commit { public_send(name).purge_later }
      else
        before_destroy { public_send(name).detach }
      end
    end
  end
end
    
```

## 上传文件
从上面has\_one\_attached源码能看出，上传文件调用了 `#{name}.attach(attachable)` 这句,所以我们从这句开始看

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/lib/active_storage/attached/one.rb
module ActiveStorage
  class Attached::One < Attached
    def attach(attachable)
      blob_was = blob if attached?
      
      # 先调父类的 create_blob_from 负责创建blob对象
      blob = create_blob_from(attachable)
      
      # 省略...
    end
  end
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/lib/active_storage/attached.rb
module ActiveStorage
  class Attached
  private
      def create_blob_from(attachable)
        case attachable
        when ActiveStorage::Blob
          attachable
        when ActionDispatch::Http::UploadedFile, Rack::Test::UploadedFile
        
          # 再 create_after_upload!, 参数io就是文件的二进制内容
          ActiveStorage::Blob.create_after_upload! \
            io: attachable.open,
            filename: attachable.original_filename,
            content_type: attachable.content_type
            
        when Hash
          ActiveStorage::Blob.create_after_upload!(attachable)
        when String
          ActiveStorage::Blob.find_signed(attachable)
        else
          nil
        end
      end
  end
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/app/models/active_storage/blob.rb
class ActiveStorage::Blob < ActiveRecord::Base
  def create_after_upload!(io:, filename:, content_type: nil, metadata: nil, identify: true)
  
    # 再是build_after_upload, 然后save!自己这个blob model
    build_after_upload(io: io, filename: filename, content_type: content_type, metadata: metadata, identify: identify).tap(&:save!)
    
  end
  
  def build_after_upload(io:, filename:, content_type: nil, metadata: nil, identify: true)
    new.tap do |blob|
      blob.filename     = filename
      blob.content_type = content_type
      blob.metadata     = metadata

      # 再blob#upload
      blob.upload(io, identify: identify)
    end
  end
  
  has_secure_token :key
  def key
    self[:key] ||= self.class.generate_unique_secure_token
  end
  
  def upload(io, identify: true)
    self.checksum     = compute_checksum_in_chunks(io)
    self.content_type = extract_content_type(io) if content_type.nil? || identify
    self.byte_size    = io.size
    self.identified   = true

    # 负责存储文件
    # 参数key是用上面has_secure_token生成的，代表文件的唯一标识
    # service是之前在config/environments/development.rb 选择的存储方式，即本地存储 disk_service
    service.upload(key, io, checksum: checksum)
  end
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/lib/active_storage/service.rb
module ActiveStorage
  class Service
    def upload(key, io, checksum: nil)
      raise NotImplementedError
    end

    def download(key)
      raise NotImplementedError
    end
  end
end

# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/lib/active_storage/service/disk_service.rb
module ActiveStorage
  class Service::DiskService < Service
    # 实现父类 NotImplementedError的方法
    # 其他的存储方式如亚马逊云存储，谷歌云存储，也是继承这个父类，只要能实现各自的upload，download方法
    # 你也可以实现自己的service类，只要有upload, download等方法。例如ruby-china上就有人为阿里云存储，七牛云存储写了service gem
    # https://ruby-china.org/topics/34445
    # https://ruby-china.org/topics/37717
    def upload(key, io, checksum: nil)
      instrument :upload, key: key, checksum: checksum do
      
        # 直接copy复制文件内容到本地，完成存储
        # make_path_for根据key，生成本地唯一的文件名以供存储
        IO.copy_stream(io, make_path_for(key))
      
        ensure_integrity_of(key, checksum) if checksum
      end
    end
  end
end
```

## 下载文件
之前提到生成的链接只有5分钟的时效，要是图片还好，5分钟内肯定能获取到图片内容。但如果是视频，那就意味着你看了5分种后，点了暂停，再点继续，浏览器重新访问视频地址时报错404，导致只能看5分钟。

解决方案：要么设置大一些的链接失效时间; 要么就不用 `rails_blob_path` 这个rails自带的方法获取下载地址，而是用自己写一个生成下载地址的方法，同时也就意味着，要自己写一个controller来负责下载文件，例如ruby-china就有[例子](https://ruby-china.org/topics/37717)

下面开始看源码，从rails\_blob\_path开始

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/config/routes.rb
Rails.application.routes.draw do

  # rails_blob_path 路径对应rails_service_blob
  direct :rails_blob do |blob, options|
    route_for(:rails_service_blob, blob.signed_id, blob.filename, options)
  end
  
  # rails_service_blob 对应blobs#show 这个action
  get "/rails/active_storage/blobs/:signed_id/*filename" => "active_storage/blobs#show", as: :rails_service_blob
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/app/controllers/active_storage/blobs_controller.rb
class ActiveStorage::BlobsController < ActiveStorage::BaseController
  include ActiveStorage::SetBlob

  def show
    expires_in ActiveStorage::Blob.service.url_expires_in
    
    # 又重定向到serivce_url方法
    # 就是这个方法产生的临时5分钟地址
    redirect_to @blob.service_url(disposition: params[:disposition])
  end
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/app/models/active_storage/blob.rb
class ActiveStorage::Blob < ActiveRecord::Base
  def service_url(expires_in: service.url_expires_in, disposition: :inline, filename: nil, **options)
    filename = ActiveStorage::Filename.wrap(filename || self.filename)

    # 再调用service#url方法。参数expires_in 就是默认5分钟的失效时间
    service.url key, expires_in: expires_in, filename: filename, content_type: content_type,
      disposition: forcibly_serve_as_binary? ? :attachment : disposition, **options
  end
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/lib/active_storage/service/disk_service.rb
module ActiveStorage
  class Service::DiskService < Service
    def url(key, expires_in:, filename:, disposition:, content_type:)
      instrument :url, key: key do |payload|
        # 生成只有5分钟时效的verified_key
        # ActiveStorage.verifier 是 ActiveSupport::MessageVerifier 的实例。用来加密，解密 key
        verified_key_with_expiration = ActiveStorage.verifier.generate(key, expires_in: expires_in, purpose: :blob_key)

        # 把verified_key作为参数，传入 rails_disk_service_url, 生成下载路径
        generated_url =
          url_helpers.rails_disk_service_url(
            verified_key_with_expiration,
            host: current_host,
            filename: filename,
            disposition: content_disposition_with(type: disposition, filename: filename),
            content_type: content_type
          )

        payload[:url] = generated_url

        generated_url
      end
    end
  end
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/config/routes.rb
Rails.application.routes.draw do
  # rails_disk_service_url 对应 disk#show 这个action
  get  "/rails/active_storage/disk/:encoded_key/*filename" => "active_storage/disk#show", as: :rails_disk_service
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/app/controllers/active_storage/disk_controller.rb
class ActiveStorage::DiskController < ActiveStorage::BaseController
  include ActionController::Live
  
  def show
    # 尝试解密verified_key. 未超过5分钟，则解密成功
    if key = decode_verified_key
      response.headers["Content-Type"] = params[:content_type] || DEFAULT_SEND_FILE_TYPE
      response.headers["Content-Disposition"] = params[:disposition] || DEFAULT_SEND_FILE_DISPOSITION

      # 调用service#download方法，获取文件内容，并返回给浏览器
      disk_service.download key do |chunk|
        response.stream.write chunk
      end
    else
      head :not_found
    end
  ensure
    response.stream.close
  end
  
  private
    # 解密verified_key
    def decode_verified_key
      ActiveStorage.verifier.verified(params[:encoded_key], purpose: :blob_key)
    end
end
```

```ruby
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activestorage/lib/active_storage/service/disk_service.rb
module ActiveStorage
  class Service::DiskService < Service
    def download(key)
      if block_given?
        instrument :streaming_download, key: key do
          File.open(path_for(key), "rb") do |file|
          
            # 分次读取，一次传5M数据
            while data = file.read(5.megabytes)
              yield data
            end
          end
        end
      else
        instrument :download, key: key do
          File.binread path_for(key)
        end
      end
    end
    
    private
      # 根据key，去找对应文件地址
      def path_for(key)
        File.join root, folder_for(key), key
      end

      def folder_for(key)
        [ key[0..1], key[2..3] ].join("/")
      end
  end
end
```
