---
layout: default
title: Golang 的 JSON 使用小技巧
---

* TOC
{:toc}

有的时候上游传过来的字段是string类型的，但是我们却想用变成数字来使用。
本来用一个`json:",string"` 就可以支持了，如果不知道golang的这些小技巧，就要大费周章了。

参考文章：http://attilaolah.eu/2014/09/10/json-and-struct-composition-in-go/

```
Go开发者们请加入我们，滴滴出行平台技术部 taowen@didichuxing.com
```

# 临时忽略struct字段

```golang
type User struct {
    Email    string `json:"email"`
    Password string `json:"password"`
    // many more fields…
}
```

临时忽略掉Password字段

```golang
json.Marshal(struct {
    *User
    Password bool `json:"password,omitempty"`
}{
    User: user,
})
```

# 临时添加额外的字段

```golang
type User struct {
    Email    string `json:"email"`
    Password string `json:"password"`
    // many more fields…
}
```

临时忽略掉Password字段，并且添加token字段

```golang
json.Marshal(struct {
    *User
    Token    string `json:"token"`
    Password bool `json:"password,omitempty"`
}{
    User: user,
    Token: token,
})
```

# 临时粘合两个struct

```golang
type BlogPost struct {
    URL   string `json:"url"`
    Title string `json:"title"`
}

type Analytics struct {
    Visitors  int `json:"visitors"`
    PageViews int `json:"page_views"`
}

json.Marshal(struct{
    *BlogPost
    *Analytics
}{post, analytics})
```

# 一个json切分成两个struct

```golang
json.Unmarshal([]byte(`{
  "url": "attila@attilaolah.eu",
  "title": "Attila's Blog",
  "visitors": 6,
  "page_views": 14
}`), &struct {
  *BlogPost
  *Analytics
}{&post, &analytics})
```

# 临时改名struct的字段

```golang
type CacheItem struct {
    Key    string `json:"key"`
    MaxAge int    `json:"cacheAge"`
    Value  Value  `json:"cacheValue"`
}

json.Marshal(struct{
    *CacheItem

    // Omit bad keys
    OmitMaxAge omit `json:"cacheAge,omitempty"`
    OmitValue  omit `json:"cacheValue,omitempty"`

    // Add nice keys
    MaxAge int    `json:"max_age"`
    Value  *Value `json:"value"`
}{
    CacheItem: item,

    // Set the int by value:
    MaxAge: item.MaxAge,

    // Set the nested struct by reference, avoid making a copy:
    Value: &item.Value,
})
```

# 用字符串传递数字

```golang
type TestObject struct {
	Field1 int    `json:",string"`
}
```

这个对应的json是 `{"Field1": "100"}`

如果json是 `{"Field1": 100}` 则会报错

# 容忍字符串和数字互转

如果你使用的是jsoniter，可以启动模糊模式来支持 PHP 传递过来的 JSON。

```golang
import "github.com/json-iterator/go/extra"

extra.RegisterFuzzyDecoders()
```

这样就可以处理字符串和数字类型不对的问题了。比如

```golang
var val string
jsoniter.UnmarshalFromString(`100`, &val)
```

又比如

```golang
var val float32
jsoniter.UnmarshalFromString(`"1.23"`, &val)
```

# 容忍空数组作为对象

PHP另外一个令人崩溃的地方是，如果 PHP array是空的时候，序列化出来是`[]`。但是不为空的时候，序列化出来的是`{"key":"value"}`。
我们需要把 `[]` 当成 `{}` 处理。

如果你使用的是jsoniter，可以启动模糊模式来支持 PHP 传递过来的 JSON。

```golang
import "github.com/json-iterator/go/extra"

extra.RegisterFuzzyDecoders()
```

这样就可以支持了

```golang
var val map[string]interface{}
jsoniter.UnmarshalFromString(`[]`, &val)
```

# 使用 MarshalJSON支持time.Time

golang 默认会把 time.Time 用字符串方式序列化。如果我们想用其他方式表示 time.Time，需要自定义类型并定义 MarshalJSON。


```golang
type timeImplementedMarshaler time.Time

func (obj timeImplementedMarshaler) MarshalJSON() ([]byte, error) {
	seconds := time.Time(obj).Unix()
	return []byte(strconv.FormatInt(seconds, 10)), nil
}
```

序列化的时候会调用 MarshalJSON

```golang
type TestObject struct {
	Field timeImplementedMarshaler
}
should := require.New(t)
val := timeImplementedMarshaler(time.Unix(123, 0))
obj := TestObject{val}
bytes, err := jsoniter.Marshal(obj)
should.Nil(err)
should.Equal(`{"Field":123}`, string(bytes))
```

# 使用 RegisterTypeEncoder支持time.Time

jsoniter 能够对不是你定义的type自定义JSON编解码方式。比如对于 time.Time 可以用 epoch int64 来序列化

```golang
import "github.com/json-iterator/go/extra"

extra.RegisterTimeAsInt64Codec(time.Microsecond)
output, err := jsoniter.Marshal(time.Unix(1, 1002))
should.Equal("1000001", string(output))
```

如果要自定义的话，参见 `RegisterTimeAsInt64Codec` 的实现代码

# 使用 MarshalText支持非字符串作为key的map

虽然 JSON 标准里只支持 string 作为 key 的 map。但是 golang 通过 MarshalText() 接口，使得其他类型也可以作为 map 的 key。例如

```golang
f, _, _ := big.ParseFloat("1", 10, 64, big.ToZero)
val := map[*big.Float]string{f: "2"}
str, err := MarshalToString(val)
should.Equal(`{"1":"2"}`, str)
```

其中 `big.Float` 就实现了 MarshalText()

# 使用 json.RawMessage

如果部分json文档没有标准格式，我们可以把原始的文本信息用string保存下来。

```golang
type TestObject struct {
	Field1 string
	Field2 json.RawMessage
}
var data TestObject
json.Unmarshal([]byte(`{"field1": "hello", "field2": [1,2,3]}`), &data)
should.Equal(` [1,2,3]`, string(data.Field2))
```

# 使用 json.Number

默认情况下，如果是 interface{} 对应数字的情况会是 float64 类型的。如果输入的数字比较大，这个表示会有损精度。所以可以 `UseNumber()` 启用 `json.Number` 来用字符串表示数字。

```golang
decoder1 := json.NewDecoder(bytes.NewBufferString(`123`))
decoder1.UseNumber()
var obj1 interface{}
decoder1.Decode(&obj1)
should.Equal(json.Number("123"), obj1)
```

jsoniter 支持标准库的这个用法。同时，扩展了行为使得 Unmarshal 也可以支持 UseNumber 了。

```golang
json := Config{UseNumber:true}.Froze()
var obj interface{}
json.UnmarshalFromString("123", &obj)
should.Equal(json.Number("123"), obj)
```

# 统一更改字段的命名风格

经常 JSON 里的字段名 Go 里的字段名是不一样的。我们可以用 field tag 来修改。

```golang
output, err := jsoniter.Marshal(struct {
	UserName      string `json:"user_name"`
	FirstLanguage string `json:"first_language"`
}{
	UserName:      "taowen",
	FirstLanguage: "Chinese",
})
should.Equal(`{"user_name":"taowen","first_language":"Chinese"}`, string(output))
```

但是一个个字段来设置，太麻烦了。如果使用 jsoniter，我们可以统一设置命名风格。

```golang
import "github.com/json-iterator/go/extra"

extra.SetNamingStrategy(LowerCaseWithUnderscores)
output, err := jsoniter.Marshal(struct {
	UserName      string
	FirstLanguage string
}{
	UserName:      "taowen",
	FirstLanguage: "Chinese",
})
should.Nil(err)
should.Equal(`{"user_name":"taowen","first_language":"Chinese"}`, string(output))
```

# 使用私有的字段

Go 的标准库只支持 public 的 field。jsoniter 额外支持了 private 的 field。需要使用 `SupportPrivateFields()` 来开启开关。

```golang
import "github.com/json-iterator/go/extra"

extra.SupportPrivateFields()
type TestObject struct {
	field1 string
}
obj := TestObject{}
jsoniter.UnmarshalFromString(`{"field1":"Hello"}`, &obj)
should.Equal("Hello", obj.field1)
```
