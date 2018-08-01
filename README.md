# JSON Transform parser plugin for Fluentd

## Overview
This is a [parser plugin](http://docs.fluentd.org/articles/parser-plugin-overview) for fluentd. It is **INCOMPATIBLE WITH FLUENTD v0.10.45 AND BELOW.**

It was forker from [fluent-plugin-json-transform](https://github.com/mjourard/fluent-plugin-json-transform) plugin and has the following fixes:
- Fixing issue when you can't use the same filter plugin multiple times with different scripts.

**Explanation:** there was a sctrict rule that script class should be named as `JSONTransformer`. And when you defined a more than 1 script with different logic, anyway those classes should be named as `JSONTransformer`. As a result you have several scripts with different logic but with the same name.

**Fix:** define new parameter called `class_name` where you can define a custom class name and this is allow you to have a lot of scripts with different class names.

- Adding `params` section with key-value pairs which can be passed to user script for usage.

## Installation
```bash
gem install fluent-plugin-json-transform_ex --version 0.1.3
```

## Configuration
```
<source>
  type [tail|tcp|udp|syslog|http] # or a custom input type which accepts the "format" parameter
  format json_transform
  transform_script [nothing|flatten|custom]
  script_path "/home/grayson/transform_script.rb" # ignored if transform_script != custom
  class_name "CustomJSONTransformer" # [optional] default value is "JSONTransformer", ignored if transform_script != custom
  <params> # any parameters which will be passed to "transform" method of class_name class
    key1 value1
    key2 value2
  </params>
</source>
```

`transform_script`: `nothing` to do nothing, `flatten` to flatten JSON by concatenating nested keys (see below), or `custom` 

`script_path`: ignored if not using `custom` script. Point this to a Ruby script which implements the class from `class_name` parameter.

`class_name`: [optional] ignored if not using `custom` script. Define name of a class which is used for transformation in `script_path` script.

### Flatten script
Flattens nested JSON by concatenating nested keys with '.'. Example:

```
{
  "hello": {
    "world": true
  },
  "goodbye": {
    "for": {
      "now": true,
      "ever": false
    }
  }
}
```

Becomes

```
{
  "hello.world": true,
  "goodbye.for.now": true,
  "goodbye.for.ever": false
}
```

### Filter Option
If you want to flatten your json after doing other parsing from the original source log.
```
<filter pattern>
  @type json_transform
  transform_script [nothing|flatten|custom]
  script_path "/home/grayson/transform_script.rb" # ignored if transform_script != custom
  class_name "CustomJSONTransformer" # [optional] default value is "JSONTransformer", ignored if transform_script != custom
  <params> # any parameters which will be passed to "transform" method of class_name class
    key1 value1
    key2 value2
  </params>
</filter>
```


## Implementing transformer class

The transformer class should have an instance method `transform` which takes a Ruby hash and returns a Ruby hash. Pay attention that the name of a class should be the same as you defined in `class_name` parameter or `JSONTransformer` in case `class_name` parameter is not defined:

```ruby
# lib/transform/flatten.rb
class JSONTransformer # or any class name defined in class_name parameter
  def transform(json, params = {}) # params - [optional] passed parameters from config
    return flatten(json, "")
  end

  def flatten(json, prefix)
    json.keys.each do |key|
      if prefix.empty?
        full_path = key
      else
        full_path = [prefix, key].join('.')
      end

      if json[key].is_a?(Hash)
        value = json[key]
        json.delete key
        json.merge! flatten(value, full_path)
      else
        value = json[key]
        json.delete key
        json[full_path] = value
      end
    end
    return json
  end
end
```