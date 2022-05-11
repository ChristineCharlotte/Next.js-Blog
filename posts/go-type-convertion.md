---
title: 'Two Forms of Pre-rendering'
date: '2020-01-01'
---

go语言的int 转成time.Duration是不可以直接转的，先把int转成int64，然后int64
//go int32转int64
var i32 int = 10
i64 := int64(i32)
fmt.Println(i64, reflect.TypeOf(i64))
//go int64转int32
i6432 := int32(i64)
fmt.Println(i6432, reflect.TypeOf(i6432))
//go string到int

var stringNum = "100"
atoi,err:=strconv.Atoi(stringNum )  //使用strconv包
//go  string到int64

int64, err := strconv.ParseInt(string, 10, 64) 
 
//int到string  
string:=strconv.Itoa(int)  

#int64到string  
 

string:=strconv.FormatInt(int64,10)  
int64转成time.Duration
例如，我需要我需要休眠10秒钟：

time.Sleep(10*time.Second)
把休眠10秒的时间转换成休眠N秒钟：

var nInt64 int64 = 10#这样就可以了
 
time.Sleep(time.Duration(nInt64) * time.Second)