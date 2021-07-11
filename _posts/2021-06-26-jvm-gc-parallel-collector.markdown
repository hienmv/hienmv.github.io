---
layout: post
title: "[JVM][GC] Parallel Collector"
date: 2021-06-26 22:47:52 +0900
categories: tech

---
Bài viết gồm 4 phần:
- Nhắc lại một số khái niệm liên quan
- The Parallel Collectors


## 1. Nhắc lại một số khái niệm liên quan

GC process được trigger khi có yêu cầu cấp pháp bộ nhớ nhưng không có đủ `available memory` để cấp phát. Tức là, GC cycles không được xếp lịch để thực thi cố định theo khoảng thời gian, mà dựa trên as-needed basis.   

Garbage Collector được chia thành 3 quá trình tương ứng với 3 vùng nhớ trong Heap:
- `Minor GC`.
- `Major GC`.
- `Full GC`.

Để hiểu được cách Garbage Collector hoạt động, chúng ta cần hiểu cách các object được phân phối trong bộ nhớ mà JVM quản lý. Các object trong JVM được lưu trữ trong vùng nhớ Heap.

![](../assets/jmv-memory-heap-layout.png)

#### Young generation
- Lưu trữ các object với thời gian hoạt động nhỏ (`short-live object`).
- Được chia thành hai vùng nhớ nhỏ hơn: `eden` và `survivor space`. Vùng nhớ `survivor space` được chia thành hai nhóm nhỏ hơn là `S0` và `S1`.

Việc thu gom các object nằm ở vùng nhớ `Young generation` được gọi là `Minor GC`. 
- Các object mới được khởi tạo sẽ nằm trong vùnh nhớ `Eden`. Khi vùng nhớ này không thể cấp phát bộ nhớ cho object mới nữa, thì `Minor GC` sẽ được gọi để thực hiện.
- Sau 1 chu kỳ hoạt động của Minor GC, những object nào vẫn còn được tham chiếu tới thì sẽ được chuyển sang `survivor space`. 
- `Minor GC` liên tục theo dõi các Object ở `S0`, `S1`, và sau “nhiều” chu kỳ quét mà object vẫn còn được sử dùng thì chúng mới được chuyển sang vùng nhớ `Old generation`. Việc quyết định thế nào là nhiều thì phụ thuộc vào việc cài đặt GC.
- Khi vùng nhớ bị đầy, thay vì sử dụng cách cổ điển là `Mark-Sweep-Compact` ở trên, thì GC sẽ sử dụng cơ chế `Mark-Copy`. Điều này sẽ dẫn tới việc giảm thiểu phân mảnh vùng nhớ. 

![](../assets/mark-sweep-compact.png)

#### Older generation

Vùng nhớ này chứa các object chuyển từ `young generation` hoặc những object mà có thời gian hoạt động đủ lâu (`long-live object`. Mỗi bộ `garbage collector` sẽ định nghĩa bao nhiêu được coi là “lâu”.

Việc thu gom các object nằm ở vùng nhớ `Old generation` được gọi là `Major GC`. 

Ngoài `Minor GC` và `Major GC`, còn có một khái niệm khác là `Full GC` được định nghĩa bằng việc thu gom các object nằm cả ở vùng nhớ `Young Generation` và `Old generation`.

#### Permanent generation

Vùng nhớ này không chứa Object, nó chứa `metadata` của JVM như các thư viện Java SE, mô tả các class và các method của ứng dụng. 

GC gần như sẽ không tương tác tới vùng nhớ này.

## 2. The Parallel Collectors
- Đây chính là GC mặc định của Java 8 và các version trước đó.
- Với `Parallel GC` quá trình xử lý các `Minor` hay `Major GC` được xử lý trên nhiều Thread (multi-thread) cho nên tốc độ xử lý của nó khá nhanh.
- Khi nó hoạt động thì tất cả các application threads đều bị dừng lại (`fully STW`), và `parallel collector` sẽ sử dùng toàn bộ available CPU core để collect memory (xử lý “rác”) nhanh nhất có thể. 
- Một số `parallel collectors`:
    - `Parallel GC`: collector đơn giản nhất cho `young generation`.
    - `ParNew`: một biến thể của Parallel GC được sử dùng cùng với `CMS collector`. (giới thiệu sau).
    - `ParallelOld`: collector được sử dụng cho `old generation` (`Tenured`). 

### 2.1 Young Parallel Collections
Xảy ra khi 1 thread muốn cấp phát bộ nhớ cho 1 object vào vùng `Eden`, nhưng không có đủ bộ nhớ cần thiết đề cấp phát `TLAB` (`thread-local allocation buffers`), và JMV không thể cấp phát một fresh `TLAB` cho thread này. Điều này dẫn đến JVM không có lựa chọn khác là phải dừng toàn bộ application thread (`fully STW`) vì một khi có một thread không thể được cấp phát bộ nhớ, thì toàn bộ các thread khác sẽ không thể sử dụng. 

Một khi toàn bộ application threads bị dừng, quá trình GC ở young generation sẽ được diễn ra:
- `HotSpot` sẽ định danh các `live object` ở `Eden` và `non-empty survivor space`.
- `Parallel GC collector` chuyển các `live object` này sang `empty survivor space`.
- Sau đó, `Eden` và `survivor space` chứa các `live objects` trước đó - sẽ được đánh dấu là `empty` và tái sử dụng. Sau đó, applications threads sẽ được bắt đầu lại. 

![](../assets/young-parallel-collection-1.png)

![](../assets/young-parallel-collection-2.png)

### 2.2 Old Parallel Collections

`ParallelOld collector` (Java 8) là collector mặc định cho old generation.

`ParallelOld collector` có nhiều điểm tương đồng với Parallel GC, tuy nhiên có một điểm khác biệt là:
- `Parallel GC` là một `hemispheric evacuating collector`, tức là nó sẽ chuyển các `live objects` sang một vùng nhớ khác. (từ `Eden` chuyển sang `survivor`; từ `survivor` chuyển sang `ternued`).
- `ParallelOld` là một `compacting collector` vì nó thực thi trên 1 vùng nhớ liên tục (`ternued`), và ở step cuối cùng trong GC cycle, nó sẽ sắp xếp lại các live objects cạnh nhau để giảm thiểu phân mảnh vùng nhớ.  

![](../assets/old-parallel-collection.png)

Vì `memory space` (`young generation` vs `old generation`) có mục đích khác nhau, dẫn đến mục đích của `young collections` và `old collections` cũng khác nhau.
- `Young collections` xử lý đối với các `short-lived objects`.
- `Old collections` xử lý đối với old space (`long-live object` hoặc `large object`). 

### Limitations of Parallel Collectors
`Parallel collectors` xử lý với toàn bộ vùng nhớ mỗi lần thực thi, và cố gắng xử lý “rác” tối ưu nhất có thể. Tuy nhiên, điều này dẫn đến một số nhược điểm:
- Đầu tiên là dừng toàn bộ application threads (`fully STW`)
    - Với `young generation`, điều này không quá là vấn đề vì nó không chứa quá nhiều `live objects`, và `pause time` cho `young collection` trên một 2GB JVM (size mặc định) chỉ một vài miliseconds (thường dưới 10ms).
    - Tuy nhiên với `old generation` là một câu chuyện khác. `Old generation` thường có size lớn gấp vài lần `young generation`, điều này dẫn đến `STW` kéo dài hơn nhiều so với `young collections`.
- Thứ hai, thời gian đánh dấu (`marking time`) tỷ lệ thuận với số lượng  `live objects` ở một `region`. Số lượng `long-lived objects` (`old objects`) có thể rất lớn, điều này có thể dẫn đến `full collection` (`both young` and `old collection`). Và điều này cũng giải thích cho điểm yếu của `parallel old collection` - `STW time` tăng tuyến tính với size của heap. 

---
**Reference**
- Optimize Java, Chapter 6: Basic Garbage collection
- [Parallel collector, Oracale](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html)
- [Java Garbage Collectors – Moving to Java7 Garbage First (G1) Collector](https://www.slideshare.net/GurpreetSachdeva2/java-garbage-collectors-moving-to-java7-garbage-first-g1-collector)