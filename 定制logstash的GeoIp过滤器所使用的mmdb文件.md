### 自定义logstash geoip filter使用的mmdb文件

##### Python 源码

```
import mmdbencoder

enc = mmdbencoder.Encoder(
    4, # IP version
    32, # Size of the pointers
    'GeoLite2-City', # Name of the table
    ['en'], # Languages
    {'en': 'GeoLite2-City'}, # Description
    compat=True) # Map IPv4 in IPv6 (::abcd instead of ::ffff:abcd) to be read by official libraries
data = enc.insert_data({"subdivisions": [{"iso_code": "0000"}],"location":{"time_zone": "Asia/Shanghai","latitude": 29.5, "longitude": 87.6 }})
enc.insert_network(u'10.0.0.0/24', data)
enc.insert_network(u'123.56.15.21/24', data)
enc.write_file('chocolatediso.mmdb')
```

##### logstash 配置

```
filter {
 geoip {
  database => "chocolatediso.mmdb"
  default_database_type => "City"
  fields => ["latitude","longitude","region_code"]
  source => ["srcIp"]
  target => "srcGeo"
 }
}
```

##### 相关源码

- logstash的GeoIp过滤器源码地址：[https://github.com/logstash-plugins/logstash-filter-geoip.git](https://github.com/logstash-plugins/logstash-filter-geoip.git)

- 本文章使用Python生成mmdb文件，涉及到py-mmdb-encoder，源码地址：[https://github.com/cloudflare/py-mmdb-encoder](https://github.com/cloudflare/py-mmdb-encoder)

##### 遇到的坑

- 当logstash geoip过滤器default_database_type属性设置为City时，mmdbencoder.Encoder表名参数必须是GeoLite2-City。

> default_database_typeedit: 
This plugin now includes both the GeoLite2-City and GeoLite2-ASN databases. If database and default_database_type are unset, the GeoLite2-City database will be selected. To use the included GeoLite2-ASN database, set default_database_type to ASN.

```
private static final String CITY_LITE_DB_TYPE = "GeoLite2-City";
private static final String COUNTRY_LITE_DB_TYPE = "GeoLite2-Country";
private static final String ASN_LITE_DB_TYPE = "GeoLite2-ASN";

switch (databaseReader.getMetadata().getDatabaseType()) {
        case CITY_LITE_DB_TYPE:
        case CITY_DB_TYPE:
        case CITY_AFRICA_DB_TYPE:
        case CITY_ASIA_PACIFIC_DB_TYPE:
        case CITY_EUROPE_DB_TYPE:
        case CITY_NORTH_AMERICA_DB_TYPE:
        case CITY_SOUTH_AMERICA_DB_TYPE:
          geoData = retrieveCityGeoData(ipAddress);
          break;
        case COUNTRY_LITE_DB_TYPE:
        case COUNTRY_DB_TYPE:
          geoData = retrieveCountryGeoData(ipAddress);
          break;
        case ASN_LITE_DB_TYPE:
          geoData = retrieveAsnGeoData(ipAddress);
          break;
        case ISP_DB_TYPE:
          geoData = retrieveIspGeoData(ipAddress);
          break;
        default:
          throw new IllegalStateException("Unsupported database type " + databaseReader.getMetadata().getDatabaseType() + "");
      }
```
logstash geoip过滤器在根据IP查询geo信息时，会判断mmdb的元数据查看数据库的类型(即mmdbencoder.Encoder函数的第三个参数，如果switch找不到对应关系就会抛出异常)

- enc.insert_data中的字典数据是带嵌套的，经纬度是存在location下面，即

```
{
    "location":{
        "time_zone": "Asia/Urumqi",
        "latitude": 29.5,
        "longitude": 87.6
    }
}
```

可以参考[https://github.com/maxmind/MaxMind-DB-Reader-java](https://github.com/maxmind/MaxMind-DB-Reader-java)中的例子读取GeoIP2-City.mmdb中的信息，在信息中查找字段的嵌套关系

Maven依赖

```
<dependency>
    <groupId>com.maxmind.db</groupId>
    <artifactId>maxmind-db</artifactId>
    <version>1.2.2</version>
</dependency>
```
Java代码

``` 
File database = new File("/path/to/database/GeoIP2-City.mmdb");
Reader reader = new Reader(database);

InetAddress address = InetAddress.getByName("24.24.24.24");

JsonNode response = reader.get(address);

System.out.println(response);

reader.close();
```

结果

```
{
    "continent":
    {
        "code": "AS",
        "names":
        {
            "de": "Asien",
            "ru": "Азия",
            "pt-BR": "Ásia",
            "ja": "アジア",
            "en": "Asia",
            "fr": "Asie",
            "zh-CN": "亚洲",
            "es": "Asia"
        },
        "geoname_id": 6255147
    },
    "country":
    {
        "names":
        {
            "de": "China",
            "ru": "Китай",
            "pt-BR": "China",
            "ja": "中国",
            "en": "China",
            "fr": "Chine",
            "zh-CN": "中国",
            "es": "China"
        },
        "iso_code": "CN",
        "geoname_id": 1814991
    },
    "city":
    {
        "geoname_id": 9088338,
        "names":
        {
            "en": "Songduo",
            "zh-CN": "松多"
        }
    },
    "location":
    {
        "accuracy_radius": 50,
        "time_zone": "Asia/Urumqi",
        "latitude": 29.5,
        "longitude": 87.6
    },
    "registered_country":
    {
        "names":
        {
            "de": "China",
            "ru": "Китай",
            "pt-BR": "China",
            "ja": "中国",
            "en": "China",
            "fr": "Chine",
            "zh-CN": "中国",
            "es": "China"
        },
        "iso_code": "CN",
        "geoname_id": 1814991
    },
    "subdivisions": [
    {
        "names":
        {
            "en": "Tibet",
            "fr": "Région autonome du Tibet",
            "zh-CN": "西藏自治区"
        },
        "iso_code": "XZ",
        "geoname_id": 1279685
    }]
}
```


