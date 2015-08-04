## 开发一个logstash-filter-geoip2插件

我们以logstash-filter-geoip2的开发为例介绍一下开发一个logstash plugin的完整流程。

### 开发logstash-filter-geoip2的原因
logstash官方提供的logstash-filter-geoip使用的IP库是Maxmind公司提供的免费数据库，其中中国的IP与地区（省，市）的对应关系尤其不准，在实际应用中发现准确率 < 50%，另外IP库中没有包含运营商(ISP)信息—判断网络问题的关键信息。logstash-filter-geoip使用的geoip API 以及IP库的格式都是Maxmind公司的v1版本，在性能上无法满足要求，常常成为logstash日志解析的瓶颈。

Maxmind公司提供的geoip API v2改进了IP库的格式，升级为maxmindDB格式，此格式的IP库占用更少空间并且查询速度有数倍提升。恰巧新浪内部有准确度较高的IP库，为纯文本格式，如下：
```
# start_ip end_ip country region city isp
1.182.0.0	1.182.31.255	China	Neimeng	Huhehaote	电信
42.56.112.0	42.56.119.255	China	Liaoning	Panjin	联通
117.146.110.0	117.146.111.255	China	Xinjiang	Hetian	移通
......
```

所以为了给公司内部使用ELK的用户提供更好的IP-地区-ISP信息服务，我们将纯文本格式的IP库转换为maxmindDB格式，并开发了对应的logstash-filter-geoip2插件。

### 开始开发插件

对于本例中的开发，除了可以参考logstash 官方的filter插件示例 [logstash-filter-example](https://github.com/logstash-plugins/logstash-filter-example)外，我们还可以直接在logstash-filter-geoip插件源码的基础上做修改。

使用git clone源码到本地：

```
git clone https://github.com/logstash-plugins/logstash-filter-geoip.git
```

我们先看一下logstash-filter-geoip的目录结构:

```
$ cd logstash-filter-geoip
$ tree
.
├── CHANGELOG.md
├── CONTRIBUTORS
├── Gemfile
├── lib
│   └── logstash
│       └── filters
│           └── geoip.rb
├── LICENSE
├── logstash-filter-geoip.gemspec
├── NOTICE.TXT
├── Rakefile
├── README.md
├── spec
│   └── filters
│       └── geoip_spec.rb
└── vendor.json

5 directories, 11 files
```

源码中没有包含二进制的IP库文件，而是通过`vendor.json`来指定在哪里获取IP库，内容很简单：
```
[
    {
        "url": "http://logstash.objects.dreamhost.com/maxmind/GeoLiteCity-2013-01-18.dat.gz",
        "sha1": "15aab9a90ff90c4784b2c48331014d242b86bf82"
    },
    {
        "url": "http://logstash.objects.dreamhost.com/maxmind/GeoIPASNum-2014-02-12.dat.gz",
        "sha1": "6f33ca0b31e5f233e36d1f66fbeae36909b58f91"
    }
]
```
插件打包的过程中，IP库被下载并存储到vendor目录中。

我们需要编辑以下文件，其他文件可保持不变。

```
# 插件的所有功能逻辑代码：
./lib/logstash/filters/geoip.rb
# 插件的单元测试代码：
./spec/filters/geoip_spec.rb
# 插件的gem描述文件：
./logstash-filter-geoip.gemspec
# 指定在哪里获取IP库：
./vendor.json
```

这次我们打算在插件中调用geoip v2 的java API来从IP库中读取地区信息。首先需要解决2个问题：
（1）IP库放在哪里
vendor.json
```
[
    {
        "url": "http://www.example.com/geoip2.mmdb",
        "sha1": "......"
    }
]
```

（2）jar包放在哪里
在这里我们做了简化，直接将geoip2 Java API的jar包放到vendor目录下，作为源码的一部分，它们也可用通过与IP库同样的方式从外部获取。geoip2 Java API的jar包下载地址为：https://github.com/maxmind/GeoIP2-java/releases/download/v2.3.1/geoip2-2.3.1-with-dependencies.zip  解压后放到vendor目录下。


编辑./lib/logstash/filters/geoip.rb
Ruby调用Java API的方法如下：
```
require "java"

require_relative "../../../vendor/geoip2-2.2.1-SNAPSHOT/lib/geoip2-2.2.1-SNAPSHOT.jar"
require_relative "../../../vendor/geoip2-2.2.1-SNAPSHOT/lib/jackson-databind-2.5.3.jar"
require_relative "../../../vendor/geoip2-2.2.1-SNAPSHOT/lib/jackson-core-2.5.3.jar"
require_relative "../../../vendor/geoip2-2.2.1-SNAPSHOT/lib/maxmind-db-1.0.0.jar"
require_relative "../../../vendor/geoip2-2.2.1-SNAPSHOT/lib/jackson-annotations-2.5.0.jar"

java_import "java.net.InetAddress"
java_import "com.maxmind.geoip2.DatabaseReader"
java_import "com.maxmind.geoip2.model.CityResponse"
java_import "com.maxmind.geoip2.record.Country"
java_import "com.maxmind.geoip2.record.Subdivision"
java_import "com.maxmind.geoip2.record.City"
java_import "com.maxmind.geoip2.record.Postal"
java_import "com.maxmind.geoip2.record.Location"
```
通过`require`, `require_relative`, `java_import` 来导入需要使用的Java class，它们的具体用法详见：[calling java from jruby](https://github.com/jruby/jruby/wiki/CallingJavaFromJRuby)
。

插件的核心代码如下：
```
class LogStash::Filters::GeoIP2 < LogStash::Filters::Base
  # 从geoip2 {} 中读取相关配置
  config_name "geoip2"

  ......

  public
  def register
    if @database.nil?
      @database = ::Dir.glob(::File.join(::File.expand_path("../../../vendor/", ::File.dirname(__FILE__)),"geoip2.mmdb")).first

      if !File.exists?(@database)
        raise "You must specify 'database => ...' in your geoip filter (I looked for '#{@database}'"
      end
    end

    db_file = JavaIO::File.new(@database)
    geoip2_initialize = DatabaseReader::Builder.new(db_file).build();

    @threadkey = "geoip2-#{self.object_id}"
  end

  public
  def filter(event)
    return unless filter?(event)

    if !Thread.current.key?(@threadkey)
      db_file = JavaIO::File.new(@database)
      Thread.current[@threadkey] = DatabaseReader::Builder.new(db_file).build();
    end

    begin
      ip = event[@source]
      ip = ip.first if ip.is_a? Array
      ipAddress = InetAddress.getByName(ip)
      response = Thread.current[@threadkey].city(ipAddress)
      country = response.getCountry()
      subdivision = response.getMostSpecificSubdivision()
      city = response.getCity()
      postal = response.getPostal()
      location = response.getLocation()
      
      geo_data_hash = Hash.new()
      geo_data_hash = { "country" => country.getName(), "region" => subdivision.getName(), "city" => city.getName(), "postal" => postal.getCode(), "latitude" => location.getLatitude(), "longitude" => location.getLongitude()}
    
    rescue com.maxmind.geoip2.exception.AddressNotFoundException => e
      # Address Not Found
      return
    rescue java.net.UnknownHostException => e
      @logger.error("IP Field contained invalid IP address or hostname", :field => @field, :event => event)
      return
    rescue Exception => e
      @logger.error("Unknown error while looking up GeoIP data", :exception => e, :field => @field, :event => event)
      return
    end

    event[@target] = {} if event[@target].nil?
    geo_data_hash.each do |key, value|
      next if value.nil? || (value.is_a?(String) && value.empty?)
      if @fields.nil? || @fields.empty? || @fields.include?(key.to_s)
        if value.is_a?(String)
          # Some strings from GeoIP don't have the correct encoding...
          value = case value.encoding
            when Encoding::ASCII_8BIT; value.force_encoding(Encoding::ISO_8859_1).encode(Encoding::UTF_8)
            when Encoding::ISO_8859_1, Encoding::US_ASCII;  value.encode(Encoding::UTF_8)
            else; value
          end
        end
        event[@target][key.to_s] = value
      end
    end
    if event[@target].key?('latitude') && event[@target].key?('longitude')
      # If we have latitude and longitude values, add the location field as GeoJSON array
      event[@target]['location'] = [ event[@target]["longitude"].to_f, event[@target]["latitude"].to_f ]
    end
    filter_matched(event)
  end # def filter
end # class LogStash::Filters::GeoIP2
```

register函数读取并初始化./vendor目录中的geoip2.mmdb IP库文件。filter函数负责根据event中的ip字段的值，调用geoip2的Java API查询IP对应的地区信息，如果未匹配成功则抛出异常：`com.maxmind.geoip2.exception.AddressNotFoundException`。获取到地区信息后，将其插入`event[@target]`, target是可配置的参数，代表输出地区信息的目标field。geoiop2 的Java API的详细使用方法请见：http://maxmind.github.io/GeoIP2-java/

重命名下面的文件:
```
$ mv ./lib/logstash/filters/geoip.rb ./lib/logstash/filters/geoip2.rb
$ mv ./spec/filters/geoip_spec.rb ./spec/filters/geoip2_spec.rb
$ mv ./logstash-filter-geoip.gemspec ./logstash-filter-geoip2.gemspec
```

logstash-filter-geoip2的开发已经完成，在做测试或正式发布之前，需要先将其打成gem包。编辑logstash-filter-geoip.gemspec，配置gem的名称，版本，依赖等：
```
Gem::Specification.new do |s|

  s.name            = 'logstash-filter-geoip2'
  s.version         = '0.1.0'
  ......
  
  # Gem dependencies
  s.add_runtime_dependency "logstash-core", '>= 1.4.0', '< 2.0.0'

  s.add_development_dependency 'logstash-devutils'
end
```

生成Gem包：
```
# Build your plugin gem
$ gem build logstash-filter-awesome.gemspec
```

打包完成后，在测试之前，需要先把插件安装到logstash中，Logstash 从 v1.5开始
```
$ $LS_HOME/bin/plugin install /your/local/plugin/logstash-filter-geoip2.gem
```

logstash conf示例：
```
# test.conf
input {
	stdin {}
}
filter {
	geoip2 {
		source => "message"
		target => "geoip2"
	}
}
output {
	stdout {
		codec => rubydebug
	}
}
```

```
# 启动Logstash
$ $LS_HOME/bin/logstash -f test.conf
```
测试结果：
```
# TODO:输出结果待补充
```

TODO：发布到rubygems.org

TODO：从rubygems.org安装logstash-filter-geoip2。
