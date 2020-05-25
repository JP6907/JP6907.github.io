---
title: Google Protobuf 的使用
catalog: true
comments: true
indexing: true
header-img: ../../../../img/article_header/article_header.png
top: false
tocnum: true
date: 2020-05-25 14:50:42
subtitle: Protobuf
tags:
    - c/c++
    - 序列化
categories:
    - serialization
---

# Google Protobuf的使用

## Protobuf

数据序列化方式除了常见的JSON和XML之外，还有Google 的 ProtoBuf，它在效率、兼容性等方面非常的出色。

为什么选择Protobuf，可以看看官方文档给出的描述：

> protocol buffers 是一种语言无关、平台无关、可扩展的序列化结构数据的方法，它可用于（数据）通信协议、数据存储等。
>
> Protocol Buffers 是一种灵活，高效，自动化机制的结构数据序列化方法－可类比 XML，但是比 XML 更小（3 ~ 10倍）、更快（20 ~ 100倍）、更为简单。
>
> 可以自定义数据的结构，然后使用特殊生成的源代码轻松的在各种数据流中使用各种语言进行编写和读取结构数据。你甚至可以更新数据结构，而不破坏由旧数据结构编译的已部署程序。

简单来说，Protobuf是一种结构数据序列化方法，具有以下特点：

1. 语言无关、平台无关，支持c++、c#、Go、Java等多种语言；
2. 高效，小、快；
3. 扩展性、兼容性好，可以更新数据结构，而不影响和破坏原有旧程序；



## Protobuf的安装

Protobuf的 [Github](https://github.com/protocolbuffers/protobuf) 仓库中有详细的安装步骤，

linux系统中的c++库安装教程：https://github.com/protocolbuffers/protobuf/blob/master/src/README.md

这里需要注意一点，教程中的第一步中：

```
$ sudo apt-get install autoconf automake libtool curl make g++ unzip
```

这里默认都是安装最新版本的库，而 protobuf 中依赖的可能是旧版本的库，比如 apt-get 安装的 libtool 高于 protobuf 中指定的版本2.4.2，解决的办法为下载指定版本的安装包(http://ftp.gnu.org/gnu/libtool/)，手动安装即可。

解决完版本不一致的问题之后，可能还会出现这个问题：

```
error: Libtool library used but 'LIBTOOL' is undefined
```

解决的办法参考（不知道是不是这个原因）：

https://blog.csdn.net/fengxianghui01/article/details/102516764

注意aclocal与libtool安装目录一致:

```
 ./configure --prefix=/usr/
```

最后重新执行 ./configure，问题解决

注意如果直接在原protobof目录执行 ./configure，可能不会生效，可以将 protobof 移动一下目录，再次执行，然后就莫名其妙地好了。



## Protobuf的使用

**第一步，创建一个.proto文件**

addressbook.proto

```
syntax = "proto2";

package tutorial;

message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```

上面创建了一个地址薄，包含四个字段：name、id、email、phones，字段前面有不同的标识符：

- required：必须提供的字段
- optional：可选字段
- repeated：可以重复出现多次的字段（包括0），适合数组或Vevtor类型的字段

字段后面的数字（=1，=2）为独一无二的标识tag，一个 message 中不允许出现多个字段拥有同样的tag。



**第二步，编译 .proto 文件生成读写接口**

```
// $SRC_DIR: .proto 所在的源目录
// --cpp_out: 生成 c++ 代码
// $DST_DIR: 生成代码的目标目录
// xxx.proto: 要针对哪个 proto 文件生成接口代码

protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/xxx.proto
```

之后会在指定目录生成两个文件：

- addressbook.pb.cc
- addressbook.pb.h

h文件中提供了如下的接口：

```c++
 // name
  inline bool has_name() const;
  inline void clear_name();
  inline const ::std::string& name() const;
  inline void set_name(const ::std::string& value);
  inline void set_name(const char* value);
  inline ::std::string* mutable_name();

  // id
  inline bool has_id() const;
  inline void clear_id();
  inline int32_t id() const;
  inline void set_id(int32_t value);

  // email
  inline bool has_email() const;
  inline void clear_email();
  inline const ::std::string& email() const;
  inline void set_email(const ::std::string& value);
  inline void set_email(const char* value);
  inline ::std::string* mutable_email();

  // phones
  inline int phones_size() const;
  inline void clear_phones();
  inline const ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >& phones() const;
  inline ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >* mutable_phones();
  inline const ::tutorial::Person_PhoneNumber& phones(int index) const;
  inline ::tutorial::Person_PhoneNumber* mutable_phones(int index);
  inline ::tutorial::Person_PhoneNumber* add_phones();
```

和其它一些方法如：

- **bool IsInitialized() const**：检查所有 required 的字段是否已经被设置；
- **string DebugString() const**：返回一个人可读的message字符串，这在调试的时候非常好用；
- **void CopyFrom(const Person& from)**： 拷贝并覆盖原有 message 中的数据；
- **void Clear()**：清空数据；
- **bool SerializeToString(string* output) const**：将message序列化字节数据并储存在output中，这里序列化后并不是String，所以是不可读的
- **bool ParseFromString(const string& data)**：反序列化
- **bool SerializeToOstream(ostream* output) const**：序列化，并输出到指定 ostream
- **bool ParseFromIstream(istream* input)**：反序列化



## 一个简单的例子

这里利用上节生成的c++文件来写一个读写的测试案例

序列化一个 Message 并输出到文件：

```c++
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// This function fills in a Person message based on user input.
void PromptForAddress(tutorial::Person* person) {
  cout << "Enter person ID number: ";
  int id;
  cin >> id;
  person->set_id(id);
  cin.ignore(256, '\n');

  cout << "Enter name: ";
  getline(cin, *person->mutable_name());

  cout << "Enter email address (blank for none): ";
  string email;
  getline(cin, email);
  if (!email.empty()) {
    person->set_email(email);
  }

  while (true) {
    cout << "Enter a phone number (or leave blank to finish): ";
    string number;
    getline(cin, number);
    if (number.empty()) {
      break;
    }

    tutorial::Person::PhoneNumber* phone_number = person->add_phones();
    phone_number->set_number(number);

    cout << "Is this a mobile, home, or work phone? ";
    string type;
    getline(cin, type);
    if (type == "mobile") {
      phone_number->set_type(tutorial::Person::MOBILE);
    } else if (type == "home") {
      phone_number->set_type(tutorial::Person::HOME);
    } else if (type == "work") {
      phone_number->set_type(tutorial::Person::WORK);
    } else {
      cout << "Unknown phone type.  Using default." << endl;
    }
  }
}

// Main function:  Reads the entire address book from a file,
//   adds one person based on user input, then writes it back out to the same
//   file.
int main(int argc, char* argv[]) {
  // Verify that the version of the library that we linked against is
  // compatible with the version of the headers we compiled against.
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;

  {
    // Read the existing address book.
    fstream input(argv[1], ios::in | ios::binary);
    if (!input) {
      cout << argv[1] << ": File not found.  Creating a new file." << endl;
    } else if (!address_book.ParseFromIstream(&input)) {
      cerr << "Failed to parse address book." << endl;
      return -1;
    }
  }

  // Add an address.
  PromptForAddress(address_book.add_people());

  {
    // Write the new address book back to disk.
    fstream output(argv[1], ios::out | ios::trunc | ios::binary);
    if (!address_book.SerializeToOstream(&output)) {
      cerr << "Failed to write address book." << endl;
      return -1;
    }
  }

  // Optional:  Delete all global objects allocated by libprotobuf.
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```

从指定文件中读取数据反序列后输出：

```c++
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// Iterates though all people in the AddressBook and prints info about them.
void ListPeople(const tutorial::AddressBook& address_book) {
  for (int i = 0; i < address_book.people_size(); i++) {
    const tutorial::Person& person = address_book.people(i);

    cout << "Person ID: " << person.id() << endl;
    cout << "  Name: " << person.name() << endl;
    if (person.has_email()) {
      cout << "  E-mail address: " << person.email() << endl;
    }

    for (int j = 0; j < person.phones_size(); j++) {
      const tutorial::Person::PhoneNumber& phone_number = person.phones(j);

      switch (phone_number.type()) {
        case tutorial::Person::MOBILE:
          cout << "  Mobile phone #: ";
          break;
        case tutorial::Person::HOME:
          cout << "  Home phone #: ";
          break;
        case tutorial::Person::WORK:
          cout << "  Work phone #: ";
          break;
      }
      cout << phone_number.number() << endl;
    }
  }
}

// Main function:  Reads the entire address book from a file and prints all
//   the information inside.
int main(int argc, char* argv[]) {
  // Verify that the version of the library that we linked against is
  // compatible with the version of the headers we compiled against.
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;

  {
    // Read the existing address book.
    fstream input(argv[1], ios::in | ios::binary);
    if (!address_book.ParseFromIstream(&input)) {
      cerr << "Failed to parse address book." << endl;
      return -1;
    }
  }

  ListPeople(address_book);

  // Optional:  Delete all global objects allocated by libprotobuf.
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```

编译的时候需要链接protobuf库：

```cmake
#CMakeLists.txt
project(TestProtobuf)
set(CMAKE_CXX_STANDARD 14)
cmake_minimum_required(VERSION 3.17)

include_directories(protobuf)
aux_source_directory(protobuf DIR_SRCS)

add_executable(test_write test_write.cpp ${DIR_SRCS})
target_link_libraries(test_write protobuf)

add_executable(test_read test_read.cpp ${DIR_SRCS})
target_link_libraries(test_read protobuf)
```



> 参考：https://developers.google.com/protocol-buffers/docs/cpptutorial