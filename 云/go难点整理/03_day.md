[TOC]
## 反射

- 反射是指一类应用，它们能够自描述和自控制。

- 通过采用某种机制来实现对自己行为的描述和检测，并能根据自身行为的状态和结构，调整或修改应用所描述行为的状态和相关的语义。

- interface 和 反射

  - 变量包括：（type,value）两部分

  - type包括static type 和 concrete type, static type就是编码时看见的类型（int, string）, concrete type是runtime时系统看见的类型。

  - 类型断言是否成功。取决于变量的concrete type, 而不是static type

    ,因此，一个reader变量如果它的concrete type也实现了write方法的话，也可以被类型断言为writer.

  - 反射主要与Golang的interface类型相关（它的type是concrete type），只有interface类型才有反射一说。

  - 每个interface变量都有一个对于pair, pair中记录了实际变量的值和类型

    ```
    （value, type）
    ```

    - value是实际变量值，type是实际变量的类型。
    - interface{}的变量包含了2个指针，一个指针指向值的类型，对应concrete type, 另一个指针指向实际的值 对应value。

    - 反射就是用来检测存储在接口变量内部(值value；类型concrete type) pair对的一种机制。

    - 如果是结构体变量，还可以获取到结构体本身的信息（包括结构体的字段，方法）
    - 通过反射，可以修改变量的值，可以调用关联的方法。

### Golang 的反射 reflect

- reflect的基本功能 TypeOf 和 ValueOf
- reflect.TypeOf() 是获取pair中的type, reflect.ValueOf()获取pair中的value

- 变量，interface{}和reflect.Value是可以相互转换的。

- Type和Kind的区别

  - Type 是类型，Kind是类别，Type和Kind可能是相同的，也可能是不同的。
    - var stu Student stu 的Type 是 pkg1.Student, Kind是struct
  - 通过反射，可以让变量在interface{}和 Reflect.value之间进行转换。

  - realValue := value.Interface().(已知的类型)

- 反射可以将“反射类型对象”转换为“接口类型变量

  - reflect.value.Interface().(已知的类型)
  - 遍历reflect.Type的Field获取其Field

- 反射可以修改反射类型对象，但是其值必须是“addressable”

  - 想要利用反射修改对象状态，前提是 interface.data 是 settable,即 pointer-interface

- 通过反射可以“动态”调用方法

- 因为Golang本身不支持模板，因此在以往需要使用模板的场景下往往就需要使用反射(reflect)来实现

  

## panic and recover

[Golang: 深入理解panic and recover (ieevee.com)](https://ieevee.com/tech/2017/11/23/go-panic.html)

## 通道

几乎很少使用 range 遍历 channel, 一般是 for + select 长期运行循环, 所以 close 一般比较少用







































