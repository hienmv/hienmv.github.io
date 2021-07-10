---
layout: post
title: "Java Virtual Machine Architecture Overview"
date: 2021-01-20 22:47:52 +0900
categories: tech

---
**Bài viết gồm 2 phần**
- Định nghĩa về JVM
- Kiến trúc của JVM (OpenJDK)

---
## I. Định nghĩa
`Java virtual machine` (JVM) là một máy ảo (`virtual computer`) được định nghĩa là một tập specification - mô tả những yêu cầu mà một cài đặt cụ thể của JVM phải tuân theo. 

JVM cho phép máy tính chạy được các chương trình viết bằng ngôn ngữ Java cũng như các chương trình được viết bằng ngôn ngữ khác mà cũng được biên dịch sang Java `bytecode` (file có định dạng `.class`), ví dụ như `Scala`. 

Luồng thực thi một chương trình viết bằng Java/Scala

![](../assets/java-execute-flow.png)

#### Một số cài đặt nổi tiếng của JVM

**OpenJDK**

Là một open source, được dẫn dắt và tài trợ bởi Oracle.

OpenJDK được coi là tham chiếu cài đặt chính thức (**official reference implementation**) của JVM kể từ Java Standard Edition phiên bản 7.

**Oracle** 

Đây là cài đặt JVM nổi tiếng nhất, dựa trên OpenJDK, và được cung cấp thêm nhiều tính năng và tuỳ chọn hơn. Ví dụ, Oracle JVM cung cấp thêm các tính năng `Flight Recorder`, `Java Mission Control`, `Application Class-Data Sharing`; nó cũng có nhiều tuỳ chọn cho `Garbage Collection` hơn.

Oracle JVM cần có giấy phép của công ty Oracle để sử dụng.

Ngoài ra, còn có một số cài đặt khác như:
- **Zulu**
- **Zing**
- **J9**
- **Android**
- …

---
### II. Kiến trúc của JVM (OpenJDK)

![](../assets/jvm-architecute-overview.png)

Bao gồm 3 subsystem chính:
- **ClassLoader**
- **Runtime Data Area**
- **Execution Engine**


### 1. ClassLoader subsystem

Subsystem này có nhiệm vụ load động các class (`dynamic class loading`) vào JVM.

`ClassLoader` sẽ  load, liên kết (`linking`) và khởi tạo (`initialization`) một class file khi mà nó được tham chiếu tới lần đầu tiên trong thời gian chạy (`runtime`), mà không phải thời gian biên dịch (`compile time`). 

![](../assets/jvm-architecute-classloader.png)

#### 1.1 Loading

Có nhiệm vụ load các class vào JVM.

Gồm 3 thành phần chính: `BootStrap ClassLoader`, `Extension ClassLoader` và `Application ClassLoader`.

1. **BootStrap ClassLoader**: có độ ưu tiên cao nhất, có nhiệm vụ load các standard core class từ `rt.jar` (`bootstrap classpath`).

2. **Extension ClassLoader**: chịu trách nhiệm load các class mở rộng (extension) của standard core class mà nằm trong thư mục `$JAVA_HOME/lib/ext`

3. **Application ClassLoader**: chịu trách nhiệm load toàn bộ application-level class: các đường dẫn của các file class này được chỉ định thông qua biến môi trường `-classpath` hay command line option `-cp`

Các `ClassLoader` này sẽ tuân theo `Delegation Hierarchy Algorithm` khi load các class file:
- Khi có 1 yêu cầu load 1 class, `ClassLoader` sẽ tìm kiếm và load class đó. 
- Nếu class đó vẫn chưa được load, `ClassLoader` sẽ gửi yêu cầu tới parent `ClassLoader` của nó để tìm kiếm, và quá trình này diễn ra đệ quy. 
- Nếu sau quá trình tìm kiếm này mà không tìm được class được yêu cầu load, nó sẽ trả về 1 exception: `ClassNotFoundException`


![](../assets/jvm-loading.png)

#### 1.2 Linking

Gồm 3 quá trình: `verify`, `prepare` và `resolve`.

1. **Verify** – `Bytecode verifier` sẽ kiểm thử xem bytecode của một class có đúng structure hay không. Nếu không, nó sẽ trả về **verification error**. 
2. **Prepare** – Tạo các biến `static` cho class hoặc interface và khởi tạo chúng với giá trị mặc định.
3. **Resolve** –  Thay thế các tham chiếu symbolic memory của các tập lệnh (`instruction`, ví dụ như `anewarray`, `checkcast`, `getfield`, `getstatic`, `instanceof`, `invokedynamic`, `invokeinterface`, `invokespecial`, `invokestatic`, `invokevirtual`, `ldc`, `ldc_w`, `multianewarray`, `new`, `putfield`, và `putstatic`.. ) với các giá trị thực tế của các tham chiếu đó ở trong `Method Area`.

#### 1.3 Initialization

Đây là bước cuối cùng trong `ClassLoading`, các biến static sẽ được gán giá trị đã được chỉ định từ source code, và các static block sẽ được thực thi.

```java
class HelloWorld {
    // class/instance variable
    int sum;
  
    // static variable
    static int num = 10;
    
    // static block
    static {
        num = 100;
    }
    // static method belonging to class
    public static void main(String[] args) {
        System.out.println("Hello world!");
    }
}
```

Trong đoạn code trên,  biến num sẽ được gán giá trị 0 (default) ở step 1.2/ Linking. Ở step này, biến numsẽ được gán giá trị 10 ở dòng số 6, sau đó sẽ lại được gán giá trị 100 ở dòng số 10.

### 2. Runtime Data Area

Được chia thành 5 thành phần chính: `Method Area`, `Heap Area`, `Stack Area`, `PC Registers` và `Native Method Stacks`.

![](../assets/jvm-architecute-runtimedata.png)

#### 2.1 Method Area

- Tất cả các `class-level` data sẽ được lưu trữ ở đây, bao gồm cả các biến static. 
- Mỗi JVM sẽ chỉ có duy nhất một `method area`, và nó là `shared resource`. 

#### 2.2 Heap Area
- Tất cả các Object và biến instances (`instance variables`) của chúng và arrays sẽ được lưu trữ ở đây.
- Mỗi JVM cũng sẽ chỉ có duy nhất một `Heap Area`.
- Vì `Method Area` và `Heap Area` là `shared memory` cho nhiều thread, nên dữ liệu được lưu trữ ở đây không phải là `thread-safe`.

#### 2.3 Stack Area
- `Runtime stack` sẽ được khởi tạo riêng biệt với mỗi thread. Với mỗi lời gọi hàm (`method call`), chỉ một entry sẽ được tạo ở stack memory (`stack frame`). 
- Tất cả các biến cục bộ (`local variables`) sẽ được khởi tạo ở stack memory. 
- Vì `Stack Area` không phải là `shared resource`, nên dữ liệu lưu trữ ở đây là `thread-safe`.
- `Stack Frame` được chia thành 3 thành phần (subentities):
    - **Local Variable Array**: lưu trữ giá trị của các biến cục bộ (local variables) của các method.
    - **Operand stack**: nếu một toán tử được yêu cầu để thực thi, thì operand stack sẽ hoạt động như một runtime workspace để thực thi các toán tử đó. 
    - **Frame data**: lưu trữ tất cả data để hỗ trợ cho `constant pool resolution` hay khôi phục `stack frame` của một lời gọi hàm (`method call`) sau khi việc method này được thực thi và trả về kết quả như mong muốn (`normal method return`), hay thông tin của catch block khi xảy ra exception.  

#### 2.4 PC Registers
- `PC registers` lưu trữ các địa chỉ của các tập lệnh (`instruction`) đang được thực thi: một khi một tập lệnh được thực thi xong, `PC register` sẽ được cập nhật với tập lệnh (`instruction`) tiếp theo.
- Mỗi thread sẽ có `PC registers` riêng biệt.

#### 2.5 Native Method stacks
- Lưu trữ các thông tin của `native method`.
- Một `native method stack` riêng biệt được khởi tạo với mỗi thread.

### 3. Execution Engine

`Bytecode` được chỉ định trong `Runtime Data Area` sẽ được thực thi bởi `Execution Engine`, và `Excecution Engine` sẽ đọc `bytecode` và thực thi chúng theo từng bước/mảng (piece by piece).

![](../assets/jvm-architecute-execution-engine.png)

#### 3.1 Interpreter

`Excecution Engine` sẽ sử dụng interpreter để biến đổi bytecode sang mã máy (native code)

Mặc dù `interpreter` thông dịch `bytecode` nhanh, nhưng thực thi rất chậm. 

Một nhược điểm của `interpreter` là khi một method được gọi nhiều lần, thì mỗi lần sẽ yêu cầu tạo mới một `interpretation` (diễn dịch).

#### 3.2 Just In Time Compiler - JIT Compiler

`JIT Compiler` được sinh ra để khắc phục nhược điểm của `Interpreter`.

Khi `Execution Engine` sử dụng `Interpreter` để biến đổi bytecode và nó tìm thấy nhiều đoạn code lặp, thì nó sẽ sử dùng `JIT compiler` để biên dịch toàn bộ bytecode đó sang mã máy (native code). Đoạn mã máy này sẽ được sử dụng trực tiếp thay thế cho những lời gọi hàm mà bị lặp đi lặp lại (`repeated method call`) nhằm tăng hiệu suất của hệ thống.

`JIT Compiler` bao gồm 4 thành phần:
1. **Intermediate Code Generator** – tạo ra các mã tạm (`intermediate code`)
2. **Code Optimizer** – chịu trách nhiệm tối ưu `intermediate code` ở trên.
3. **Target Code Generator** – chịu trách nhiệm tạo ra các mã máy (native code).
4. **Profiler** – một thành phần đặc biệt, chịu trách nhiệm tìm kiếm các hotspots:  kiểm tra xem liệu rằng một method có được gọi nhiều lần hay không,..

#### 3.3 Garbage Collector
- Thu thập và xoá bỏ các object mà không còn được tham chiếu tới.
- `Garbage collection` có thể được kích hoạt (trigger) thông qua lời gọi `System.gc()`, nhưng việc thực thi `garbage collection` sẽ không được đảm bảo.

### 4. Java Native Interface (JNI)

`JNI` sẽ tương tác với các thư viện chứa các native method và cung cấp các thư viện native (`native libraries`) mà được yêu cầu bởi `Execution Engine`.

### 5 Native Method Libraries
- Là một tập các thư viện native (`native libraries`) mà được yêu cầu bởi Execution Engine.
- Các thư viện này thường được viết bằng C/C++ và assembly.



**Reference**
- [JVM, wikipedia](https://en.wikipedia.org/wiki/Java_virtual_machine)
- [JVM architecture, dzone](https://dzone.com/articles/jvm-architecture-explained)
- [JVM inside, artima](https://www.artima.com/insidejvm/ed2/jvm8.html)
- [List of JVM, wikipedia](https://en.wikipedia.org/wiki/List_of_Java_virtual_machines)