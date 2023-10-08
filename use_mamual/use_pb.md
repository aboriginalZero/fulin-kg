

属性定义分为四部分：标注 + 类型 type + 属性名 field name + 唯一标签号 tag + [默认值 ]，其示意如下：

```
optional bool alloc_even = 81 [default = false];
```

区分基本类型和对象类型

基本数据类型包括：double、float、bool、string、bytes、int32、int64、uint32、uint64、sint32、sint64、fixed32、fixed64、sfixed32、sfixed64

对于枚举类型 pb 有个约束是，枚举的第一项对应的值必须为 0。



1. optional 基本类型
   * xxx()，获取属性值；
   * set_xxx()，修改属性值。
2. optional 对象类型
   * xxx()，获取对象引用（只读）；
   * mutable_xxx()，获取对象执行（可修改）；
3. repeated 基本类型
   * xxx()，获取整个数组的引用；
   * xxx(idx)，获取指定位置的元素；
   * mutable_xxx()，获取整个数组的指针；
   * xxx_size()，获取数组大小；
   * add_xxx(val)，添加元素；
4. repeated 对象类型
   * xxx()，获取整个数组的引用；
   * xxx(idx)，获取指定位置的元素引用；
   * mutable_xxx()，获取整个数组的指针；
   * mutable_xxx(idx)，获取可修改的指定对象指针；
   * add_xxx()，获取可修改的对象指针；

```cpp
// 写入
tutorial::Person::PhoneNumber* phone_number = person->add_phone(); 
phone_number->set_number("13051889399");
phone_number->set_type(tutorial::Person::MOBILE);
// 读取
const tutorial::Person::PhoneNumber& phone_number = person.phone(j);
cout << phone_number.number() << phone_number.type();
```

repeated message 可以视为 std::vector，允许通过 [] 方法访问数组成员，另外，提供 xxx_size() 方法返回数组大小，mutable_xxx() 方法返回指针，add_xxx() 方法增加一个成员。

对于对象类型，可以使用 mutable_xxx() 获取对象指针，然后设置对象属性，也可以使用 CopyFrom() 直接拷贝，也可以使用 Swap() 进行交换。

```cpp
syntax = "proto2";

message Test {
  optional Obj obj = 1;
}

message Obj {
  optional int64 id = 1;
  optional string name = 2;
}
```

对其使用方式为

```cpp
Test test;
Obj o1;
test.mutable_obj()->CopyFrom(o1);
test.mutable_obj()->Swap(&o1);
```





string DebugString() const; 返回 message 的可阅读字符串，可用于调试程序；

对 repeated message 类型字段做大量 add 操作时（超过 1w），需要减少嵌套 message 个数，以及预分配数据空间，避免大量重复空间分配。

```cpp
int cnt = 3000;
columes* col = recs.add_cols();
// vec 属性是 repeat 类型
col->mutable_vec()->Reserve(times);
for (int i = 0; i < times; i ++) {
  col->add_vec(i);
}
```





如果为 optional，发送端没有包含该属性，则接收端在解析式采用默认值。对于默认值，如果已设置默认值，则采用默认值，如果未设置，则类型特定的默认值为使用，例如 string 的默认值为 “”。

代码中 [(zbs.labels).as_str = true] 的意思，标记一个可以被安全转换为 str 的 bytes field，对于之后向 zbs-proto 提交的新字段，除了明确需要使用 bytes 类型存放的字段，其他字符串类型都建议直接使用 “string” 关键字标记，减少不必要的 bytes 类型的使用。引入这个功能的原因是 python2 中 string 为 bytes 的 alias，但 python3 中 string 和 bytes 则完全不同。 zbs-client-py 的下游用户如果直接适配这种改变需要编写很多针对性的垃圾代码。



pb message 升级原则

1. 如果不是 id 类的字段，最好都用 optional，便于做兼容性升级；
2. 不要修改已经存在字段的标签号；
3. 新添加的字段必须是 optional / repeated 限定符，否则无法保证新老程序在互相传递消息时的消息兼容性；
4. 在原有的消息中，不能移除已经存在的 required 字段，optional 和 repeated 类型的字段可以被移除，但是他们之前使用的标签号必须被保留，不能被新的字段重用（最好是保留，注释上提示 [deprecated = true]）；
5. 如果想修改原有字段的类型，为了保证兼容性，只能修改成与原有类型兼容的类型，其中 int32、uint32、int64、uint64 和 bool 等类型之间是兼容的。



pb 反射机制

```cpp
auto set_value = [&params, &request](const std::string& field_name, uint64_t lower, uint64_t upper) {
	const auto* desp = UpdatableRepositionParams::descriptor();
  const auto* field = desp->FindFieldByName(field_name);
  CHECK(field && (field->is_required() || field->is_optional()));

  const auto* reflection = request->GetReflection();
  CHECK(reflection->HasField(params, field));

  auto limit = reflection->GetUInt64(params, field);
  if (limit < lower || limit > upper) {
    return Status(EBadArgument) << "The given " << field_name << " should be in range [" 
      													<< lower << ", " << upper << "].";
  }
  
  reflection->SetUInt64(params.mutable_updatable_reposition_params(), field, limit);
  return Status::OK();
};

set_value("static_generate_cmds_per_round_limit", kMinGenerateCmdsPerRound, kMaxGenerateCmdsPerRound)
```

* https://www.cnblogs.com/bwbfight/p/14994494.html
* https://blog.51cto.com/u_5650011/5389330



### 官方示例

[官方例子](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.service#RpcChannel.CallMethod.details)

引入头文件以及使用命名空间

```c++
#include <google/protobuf/service.h>
namespace google::protobuf
```

自定义 Service 

```c++
service MyService {
  rpc Foo(MyRequest) returns(MyResponse);
}
```

通过编译自动生成待服务端实现的接口 MyService 和客户端使用的类 MyService::Stub

RPC 服务端实现该接口

```c++
class MyServiceImpl : public MyService {
public:
		MyServiceImpl() {}
  	~MyServiceImpl() {}

    // implements MyService ---------------------------------------

  	void Foo(google::protobuf::RpcController* controller,
           const MyRequest* request,
           MyResponse* response,
           Closure* done) {
    	// ... read request and fill in response ...
    	done->Run();
  }
};
```

在 RPC 客户端调用该远程服务

```c++
MyRpcChannel channel("rpc:hostname:1234/myservice");
MyRpcController controller;
MyServiceImpl::Stub stub(&channel);
FooRequest request;
FooResponse response;

// ... fill in request ...

stub.Foo(&controller, request, &response, NewCallback(HandleResponse));
```

### protobuf 介绍

protobuf 用于序列化操作，相较于XML、JSON、YAML，整体来说，Protobuf序列化和反序列的性能都是比较高的，编码后的数据大小也不错。

01 年在 Google 内部出现 proto1, 08 年把 Protobuf 开源出来，16 年推出 proto3，但目前仍支持 proto2。

定义 proto 文件

```protobuf
syntax = "proto2";					// 行指定 protobuf 的版本
package tutorial;					// 同名的 message 可以通过 package 加以区分
message Person {
    required string name = 1;		// required 必须字段
    required int32 id = 2;
    optional string email = 3;		// optional 可选字段
    enum PhoneType {
        HOME = 1;
        WORK = 2;
    }
    message PhoneNumber {
        required string number = 1;
        optional PhoneType type = 2 [default = HOME];
    }
    repeated PhoneNumber phones = 4;	// repeated 类似于数组
}
message AddressBook {
    repeated Person people = 1;
}
```

使用 IDL 把 proto 文件编译成 c++ 代码

```shell
protoc -cpp_out= . myname.proto
```

在 c++ 使用 proto 对象

```c++
#include <iostream>
#include <fstream>
#include <string>
#include "myname.proto.pb.h"
using namespace std;

void WriteIntoAddressBook(tutorial::Person *person) {
	cout << "Enter person ID number: ";
    int id, cin >> id, person->set_id(id);
    cin.ignore(256, '\n');	//忽略最后的回车

    cout << "Enter name: ";
    getline(cin, *person->mutable_name());

    cout << "Enter email address (blank for none): ";
    string email, getline(cin, email);
    // email 字段可选，所以可能为空，不能直接用getline(cin, *person->mutable_email());
    if (!email.empty()) {	
        person->set_email(email);
    }
    
    while (true) {
        cout << "Enter a phone number (or leave blank to finish): ";
        string number, getline(cin, number);
        if (number.empty()) {
            break;
        }
        // repeated 字段用 add_xxx(), 一般字段用 set_xxx()
        tutorial::Person::PhoneNumber *phone_number = person->add_phones();
        phone_number->set_number(number);
        cout << "Is this a mobile, home, or work phone? ";
        string type, getline(cin, type);
        if (type == "home") {
            phone_number->set_type(tutorial::Person::HOME);
        } else if (type == "work") {
            phone_number->set_type(tutorial::Person::WORK);
        } else {
            cout << "Unknown phone type.  Using default." << endl;
        }
    }    
}

void ReadFromAddressBook(const tutorial::AddressBook &address_book) {
    for(int i = 0; i < address_book.people_size(); i ++) {
        const tutorial::Person &person = address_book.people(i);
        cout << "Person ID: " << person.id() << endl;
        cout << "  Name: " << person.name() << endl;
        // 可选字段先判断有无
        if (person.has_email()) {
            cout << "  E-mail address: " << person.email() << endl;
        }
        for (int j = 0; j < person.phones_size(); j++) {
            const tutorial::Person::PhoneNumber &phone_number = person.phones(j);
            switch (phone_number.type()) {
                case tutorial::Person::HOME:
                    cout << "  Home phone #: ", break;
                case tutorial::Person::WORK:
                    cout << "  Work phone #: ", break;
            }
            cout << phone_number.number() << endl;
        }
    }
}

int main() {
	GOOGLE_PROTOBUF_VERIFY_VERSION;
	tutorial::AddressBook address_book;
	// 这里暂时不考虑各种错误异常处理，仅表示从文件中读取原有的 address_book
    {
    	fstream input("/my_input_path", ios::in | ios::binary);
    	address_book.ParseFromIstream(&input);
    }
    ReadFromAddressBook(address_book);
    WriteIntoAddressBook(address_book.add_people());
    {
    	fstream output("my_output_path", ios::out | ios::trunc | ios::binary);
        address_book.SerializeToOstream(&output);
    }
    google::protobuf::ShutdownProtobufLibrary();
    return 0;
}
```

编译

```shell
g++ addressbook.pb.cc main.cpp -o main -lprotobuf
```



