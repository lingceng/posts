## Rails项目连接第二个数据库小结
业务需要，Rails项目需要接入第二个数据库，这里记录一些适配的细节。
环境：Rails 4.2.5 Ruby2.3.0 Mysql5.7.10

大概这几部分内容：
+ 不同的DB配置
+ 不按约定的表名
+ 不同的updated_at或者created_at名字
+ 存储的时间是不同的时区
+ bit做为boolean

### 不同的DB配置
在 config/database.yml 添加第二个数据库的配置，添加环境做后缀，这样方便测试和上线。

    data_source_development: &data_default
      adapter: mysql2
      encoding: utf8mb4
      reconnect: true
      database: data_source_development
      pool: 5
      username: root
      password: password
      host: localhost

    data_source_test:
      <<: *data_default
      database: data_source_test

    data_source_production:
      <<: *data_default
      database: data_source_production

创建一个继承自ActiveRecord::Base的类，设置为abstract_class

    class DataRecord < ActiveRecord::Base
      self.abstract_class = true
      establish_connection("data_source_#{Rails.env}".to_sym)
    end

### 不按约定的表名
Rails里表名需要是类名称下划线形式的复数，我接入的第二个数据库全是单数，还有前缀。
例如 Data::Player 需要对应到 data_player 表。需要做下面的配置：

    class DataRecord < ActiveRecord::Base
      self.abstract_class = true
      # New added
      self.table_name_prefix = "data_"
      # New added
      self.pluralize_table_names = false
      establish_connection("data_source_#{Rails.env}".to_sym)
    end

如果没有规律，可以通过`self.table_name = 'table_name'`来设置表名。

### 不同的updated_at或者created_at
第二个数据库不是用updated_at来表示更新时间，而是用last_updated，可以通过重写下面的方法设置:

    def timestamp_attributes_for_update
      [:last_updated]
    end

创建时间用 timestamp_attributes_for_create

### 存储的时间使用不同的时区
Rails在存储和获取 datatime 时会根据配置做转换，当前的配置是存储utc，显示北京时区。

    config.time_zone = 'Beijing'
    config.active_record.default_timezone = :utc

第二个数据库存储的是北京时间，所以需要处理第二个数据库的时区转换。
Rails不能对时区做局部配置，所以处理麻烦一些，我使用的是在写入时+8，读取时-8。例如

     def start_time
       attributes['start_time'] && (attributes['start_time'] - 8.hour)
     end

     def start_time=(value)
        converted = if value.present?
                      (value.is_a?(String) ? Time.zone.parse(value) : value) + 8.hour
                    else
                      value
                    end
        super(attr, converted)
     end

当然这样每一个model重写有点麻烦，所以用了个[Concern](https://gist.github.com/lingceng/2bf51609987ef15390fc751e2863202f)，检查并重写所有datetime类型字段。

### bit表达boolean
Rails里的boolean类型在MySQL里用`tynyint(1)`，接入的第二个数据库是用的`bit(1)`。
我用了下面的方式做转换。

    def hidden
      read_attribute(:hidden) == "\x01"
    end

    def hidden=(value)
      write_attribute(:hidden, ActiveRecord::Type::Boolean.new.type_cast_from_user(value).presence && "\x01")
    end

### 其他
时区的转换有些繁琐，希望能找到更简洁的办法。
