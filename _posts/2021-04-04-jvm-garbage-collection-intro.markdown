---
layout: post
title: "[JVM] Garbage Collection Introduction"
date: 2021-04-04 22:47:52 +0900
categories: tech

---
Bài viết 4 gồm:
- Định nghĩa Garbage collection
- Tradeoffs and Pluggable Collectors
- Garbage Collection và JVM Memory Model.
- Default Garbage collection implementation

#### Giới thiệu và nhắc lại một số khái niệm liên quan

Java virtual machine (JVM) là một máy ảo (virtual computer) được định nghĩa là một tập specification - mô tả những yêu cầu mà một cài đặt cụ thể của JVM phải tuân theo.

JVM cho phép máy tính chạy được các chương trình viết bằng ngôn ngữ Java cũng như các chương trình được viết bằng ngôn ngữ khác mà cũng được biên dịch sang `Java bytecode` (file có định dạng `.class`), ví dụ như Scala.

`HotSpot` là một cài đặt của JVM, và được phát triển bởi Oracle. OpenJDK là một project mà chứa open-source cài đặt của HotSpot.

Bài viết này giới thiệu về Garbage collection của HotSpot.

---
## 1. Định nghĩa Garbage collection

Garbage collection là một tiến trình mà JVM sử dụng để quản lý bộ nhớ một cách tự động khi thực thi một chương trình. Chương trình đó có thể viết bằng Java, Scala hay các ngôn ngữ khác mà được thực thi trên nền JVM.

Garbage Collector (GC) sẽ tự động quét dọn bộ nhớ. Vì vậy lập trình viên không cần quan tâm đến việc quản lý bộ nhớ hay là phân phối lại bộ nhớ (allocate memory) của chương trình. Điều này sẽ giúp giảm thiểu các lỗi liên quan tới bộ nhớ như `memory leak` hay là `out of memory`.

Garbage collection có 2 nguyên tắc cơ bản mà bất kỳ cài đặt nào của nó cũng phải tuân theo:
- Thuật toán phải thu gom toàn bộ các object mà không còn được tham chiếu tới.
- Không được thu gom bất kỳ object nào mà chúng vẫn đang được tham chiếu tới.

---
## 2. Tradeoffs and Pluggable Collectors
Mặc dù Java platform có một garbage collector (GC) mặc định, nhưng specification của ngôn ngữ Java và JVM không nói rõ một GC sẽ cần được cài đặt như thế nào. 

Trong Oracle environment, GC subsystem được coi như một pluggable subsystem, tức là một chương trình Java có thể thực thi với các GC khác nhau mà không cần được viết lại, mặc dù hiệu năng của chương trình có thể khác nhau khi thực thi với các GC khác nhau. 

Mục đích chính cho việc có pluggable GC đó là GC là một kỹ thuật tính toán chung. (`general computing technique`), và một GC algorithm có thể không phù hợp với tất cả các trường hợp/bài toán. 

Một số yếu tố cần cân nhắc khi chọn sử dụng một GC algorithm:
- `Pause time` (aka pause length or duration): khoảng thời gian của một GC cycle mà khi đó các Application threads bị tạm dừng.
- `Throughput` (as a percentage of GC time to application runtime).
Pause frequency (how often the collector needs to stop the application).
- `Reclamation efficiency` (how much garbage can be collected on a single GC duty cycle).
- `Pause consistency` (are all pauses roughly the same length?)

---
## 3. Garbage Collection và JVM Memory Model

Garbage collector thực thi bất cứ khi nào cần thiết mà không phải được thực thi sau một khoảng thời gian cố định (regular interval).

Một thuật toán cổ điển đối khi nhắc tới Garbage collection đó là `mark-and-sweep`.
- Sử dụng 1 danh sách để quản lý các pointer tới từng object mà đã được cấp phát. 
- Ý tưởng thực thi: 
    - Duyệt qua danh sách trên và xoá các bit mark của nó đi.
    - Tìm các object mà vẫn còn được tham chiếu (live object), và đánh dấu object vẫn còn được tham chiếu tới (mark bit).  
    - Duyệt lại danh sách trên, và đối với các object mà nó không được đánh dấu:
        - Lấy lại vùng nhớ trên heap của object đó và đặt nó vào danh sách vùng nhớ có sẵn (free list).
        - Loại bỏ object đó ra khỏi danh sách trên.
![](../assets/mark-sweep.png)

Garbage Collector được chia thành 3 quá trình tương ứng với 3 vùng nhớ trong Heap:
- Minor GC.
- Major GC.
- Full GC.

Để hiểu được cách Garbage Collector hoạt động, chúng ta cần hiểu cách các object được phân phối trong bộ nhớ mà JVM quản lý.

Các object trong JVM được lưu trữ trong vùng nhớ Heap.

![](../assets/jmv-memory-heap-layout.png)

### 3.1 Young generation
- Lưu trữ các object với thời gian hoạt động nhỏ (`short-live object`).
- Được chia thành hai vùng nhớ nhỏ hơn: `eden` và `survivor space`. Vùng nhớ `survivor space` được chia thành hai nhóm nhỏ hơn là `S0` và `S1`.

Việc thu gom các object nằm ở vùng nhớ `Young generation` được gọi là `Minor GC`. 
- Các object mới được khởi tạo sẽ nằm trong vùnh nhớ `Eden`. Khi vùng nhớ này không thể cấp phát bộ nhớ cho object mới nữa, thì `Minor GC` sẽ được gọi để thực hiện.
- Sau 1 chu kỳ hoạt động của Minor GC, những object nào vẫn còn được tham chiếu tới thì sẽ được chuyển sang `survivor space`. 
- `Minor GC` liên tục theo dõi các Object ở `S0`, `S1`, và sau “nhiều” chu kỳ quét mà object vẫn còn được sử dùng thì chúng mới được chuyển sang vùng nhớ `Old generation`. Việc quyết định thế nào là nhiều thì phụ thuộc vào việc cài đặt GC.
- Khi vùng nhớ bị đầy, thay vì sử dụng cách cổ điển là `Mark-Sweep-Compact` ở trên, thì GC sẽ sử dụng cơ chế `Mark-Copy`. 
    > Mark-Sweep-Compact có 3 step, và sau step 3 (compact), các live object sẽ được copy để chúng nằm cạnh nhau nhằm giảm phân mảnh vùng nhớ. Tuy nhiên, nó có nhược điểm là làm tăng thêm thời gian pause time của GC cycle. 
    Mark-copy tương tự như mark-sweep-compact, nhưng chỉ gồm 2 step, Điều này có nghĩa thời gian xử lý sẽ ngắn hơn so với Mark-Sweep-Compact. Nhược điểm là nó cần thêm một vùng nhớ nữa (more memory region).


![](../assets/mark-sweep-compact.png)

### 3.2 Older generation

Vùng nhớ này chứa các object chuyển từ `young generation` hoặc những object mà có thời gian hoạt động đủ lâu (`long-live object`. Mỗi bộ `garbage collector` sẽ định nghĩa bao nhiêu được coi là “lâu”.

Việc thu gom các object nằm ở vùng nhớ `Old generation` được gọi là `Major GC`. 

Ngoài `Minor GC` và `Major GC`, còn có một khái niệm khác là `Full GC` được định nghĩa bằng việc thu gom các object nằm cả ở vùng nhớ `Young Generation` và `Old generation`.

### 3.3 Permanent generation

Vùng nhớ này không chứa Object, nó chứa `metadata` của JVM như các thư viện Java SE, mô tả các class và các method của ứng dụng. 

GC gần như sẽ không tương tác tới vùng nhớ này.

**Vấn đề gặp phải trong quá trình Minor GC và Major GC**
- Trong quá trình thực thi `Minor GC` và `Major GC`, `STW` (`Stop-the-world`) sẽ diễn ra. `STW` có nghĩa là tất cả các application thread sẽ bị dừng lại, cho tới khi GC thực hiện xong. Điều này ảnh hưởng tới performance của chương trình.
- `STW` diễn ra ở thời điểm: sau khi quá trình `Minor GC`, khi các object vẫn còn được tham chiếu thì sẽ được chuyển từ `Young generation` sang `Old generation`; hoặc từ `Eden` sang `S0`; hoặc từ `S0` sang `S1`.

---
## 4. Default Garbage collection implementation

### Parallel Garbage Collector
- Đây chính là GC mặc định của Java 8.
- Với `Parallel GC` quá trình xử lý các `Minor` hay `Major GC` được xử lý trên nhiều Thread (multi-thread) cho nên tốc độ xử lý của nó khá nhanh. 
- Khi nó hoạt động thì các thread khác của chương trình sẽ bị dừng lại, điều này vẫn gây ảnh hưởng tới hệ thống.

### Garbage First Collector
- Là GC mặc định của Java 9, 10 và Java 11.
- `G1` được ra đời để quản lý các vùng `HEAP > 4G` hiệu quả hơn. 
- Khác với các GC khác `G1` chia vùng nhớ HEAP thành các phần nhỏ hơn ( có dung lượng từ 1 đến 32MB), khi GC hoạt động GC sẽ đánh dấu (marking) vùng nhớ nào có nhiều "rác" nhất, từ đó sẽ dọn dẹp (sweeping) vùng nhớ đó đầu tiên và sẽ thực hiện việc dồn bộ nhớ ngay lúc đó, có nghĩa là G1 vừa thực hiện dọn rác và dồn bộ nhớ đồng thời.

---
**Next sprint**
- Concurrent GC Theory
- Chi tiết từng implementation của GC
    - Parallel Garbage Collector
    - Garbage First collector.
    - Concurrent Mark and Sweep collector.

**Sprint after next**
- GC logging, monitoring, and Tuning.

**Reference**
- Optimize Java, Chapter 6: Basic Garbage collection
- [garbage collection algorithms, plumbr.io](https://plumbr.io/handbook/garbage-collection-algorithms)
- [garbage collector, batnamv](https://batnamv.medium.com/garbage-collector-231d6c327b08)
- [minor/major/full gc, dzome](https://dzone.com/articles/minor-gc-vs-major-gc-vs-full)

