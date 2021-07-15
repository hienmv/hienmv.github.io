---
layout: post
title: "[JVM][GC] Garbage-First collector"
date: 2021-07-14 22:47:52 +0900
categories: tech

---
>Bài viết gồm 2 phần:
>- G1 Heap Layout and Regions.
>- G1 Algorithms Design.
>- G1 Phases.
>- Recommended Use Cases for G1.

---
Garbage-First (G1) collector là GC collector khác rất nhiều so với CMS hay parallel collector, được thiết kế hướng tới những thiết bị multi-processor với lượng memory lớn, và đáp ứng được các yêu cầu như: (GC) pause time goals with a high probability; high throughput.

G1 collector là default collector từ Java 9, thay thế cho parallel collectors.

G1 được thiết kế cho các ứng dụng như: 
- Có thể thực thi đồng thời applications threads và GC cycle giống như CMS collector.
- Thu gom, sắp xếp lại những vùng nhớ không còn được sử dụng (Compact free space) mà không kéo dài GC pause.
- Cần nhiều hơn `predictable GC pause duration`.
- Không muốn hi sinh quá nhiều hiệu suất về throughput.
- Không yêu cầu vùng nhớ Heap có dung lượng quá lớn. 

G1 được thiết kế để thay thế cho CMS.
- Sự khác biệt lớn nhất giữa G1 và CMS là: G1 là `compacting collector`, điều này dẫn tới một kết quả là G1 sẽ giúp giảm thiểu tối đa vấn đề phân mảnh vùng nhớ. 
- G1 cũng hỗ trợ nhiều `predictable GC pause duration` hơn CMS, và cho phép người dùng chỉ định `pause time target`.

---
## 1. G1 Heap Layout and Regions
Đối với những Garbage collectors như `serial`, `parallel`, `CMS`, vùng nhớ heap được chia làm 3 phần với size được chỉ định trước: `young generation`, `old generation`, `permanent generation` 

![](../assets/old-memory-layout.png)


G1 collector sử dụng một hướng tiếp cận khác:

![](../assets/g1-memory-layout.png)

Heap được chia thành các region có cùng kích thước, và mỗi region là một vùng nhớ ảo (virtual memory) liên tiếp nhau.
Các region được chỉ định cùng các role (`eden`, `survivor`, `old`) như những collector khác, nhưng kích thước của chúng có thể chỉ định được.
- Mặc định kích thước của 1 region này là 1MB, (sẽ lớn hơn đối với heap có kích thước lớn hơn). G1 cho phép các region có kích thước là 1, 2, 4, 8, 16, 32, 64 MB.
- Công thức tính số lượng regions: 
  ```
  Number of regions = <Heap size> / <region size>
  ```
- Object được chuyển qua giữa các regions trong quá trình collection.
- Nếu một Object có dung lượng lớn hơn 1/2 kích thước của 1 region thì nó sẽ được cấp phát tại vùng nhớ `Humongous region`. (là vùng nhớ đang free và liên tục) và sau khi cấp phát thì trở thành 1 phần của `old generation`.

---
## 2. G1 Algorithms Design 

High-level của G1 algorithms: 
- Uses a concurrent marking phase: 
  - G1 tiến hành đánh dấu các live objects đồng thời với application threads.
- Is an evacuating collector: 
  - chuyển các object từ `eden` sang `survivor space`, rồi tới `Tenured` (`old generation`) (tương tự như các collector khác)
- Provides “statistical compaction”: 
  - Khi khởi động, G1 sẽ thống kê có bao nhiêu regions mà có thể thu thập ở mỗi GC cycle. Nếu lượng memory được thu thập cân băn với số lượng object mới cần cấp phát kể từ lần GC cycle trước đó, thì với những object mới này, G1 sẽ không cần cấp phát thêm vùng nhớ mới nữa, mà sẽ dùng lại vùng nhớ vừa mới được thu thập. 


**Recal: Weak Generational Hypothesis**
1. Sự phân phối vòng đời của objects (object lifetime) ở JVM đó là: phần lớn các object là short-lived objects (young objects) và phần còn lại là là long-lived object. (old objects)
2. Có một lượng nhỏ reference từ old objects tới young objects.

Với nội dung `2.` ở trên:
- Đối với parallel và CMS collectors, Hotspot sử dụng một mechanism `card tables` để lưu lại old objects nào có thể refer tới young objects nào.
  > card table là 1 array of byte được quản lý bởi JVM, mỗi phần tử trong array tương đương với 1 vùng nhớ 512-byte trong old-generation space. Mỗi khi 1 field (of reference type) của 1 old object bị thay đổi, thì entry trong card table mà chứa `instanceOop` tương ứng sẽ bị đánh dấu là `dirty`, với `instanceOop` là represent instances của 1 Java class tại thời điểm runtime.

- G1 collector cũng có 1 feature tương tự để tracking liên kết (reference) giữa các region.
  - `Remembered Sets` (`RSets`) được sử dụng để tracking object reference tới 1 region. Mỗi region sẽ có 1 `RSets`. Điều này có nghĩa, với 1 region A, thay vì tracking reference tới A bằng cách duyệt qua toàn bộ heap, G1 chỉ cần kiểm tra trên `RSets`, và sau đó scan những regions nào đang reference tới A. Điều này sẽ giúp cho việc chạy song song và độc lập GC ở mỗi region. 
   ![](../assets/g1-rset.png)
   - Ngoài ra, ở G1 còn có `Collection Sets` (`CSets`) - danh sách các region sẽ được collect ở 1 GC. Toàn bộ live object ở 1 `CSet` sẽ được copy/move trong quá trình GC.Những danh sách các region này có thể là Eden, survivor, hoặc old generation.
    ![](../assets/g1-cset.png)

---
## 3. G1 Phases
Gồm 5 phase:
```
1. Initial Mark (STW)
2. Concurrent Root Scan 
3. Concurrent Mark
4. Remark (STW)
5. Cleanup (STW)
```

## 4. Recommended Use Cases for G1

The first focus of G1 is to provide a solution for users running applications that require large heaps with limited GC latency. This means heap sizes of around 6GB or larger, and stable and predictable pause time below 0.5 seconds.

Applications running today with either the CMS or the ParallelOldGC garbage collector would benefit switching to G1 if the application has one or more of the following traits.
- Full GC durations are too long or too frequent.
- The rate of object allocation rate or promotion varies significantly.
- Undesired long garbage collection or compaction pauses (longer than 0.5 to 1 second)

Note: If you are using CMS or ParallelOldGC and your application is not experiencing long garbage collection pauses, it is fine to stay with your current collector. Changing to the G1 collector is not a requirement for using the latest JDK.

---
**Reference**
- Optimize Java, Chapter 7: Advance Garbage collection
- [G1 collector, Oracale](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)
- [Java Garbage Collectors – Moving to Java7 Garbage First (G1) Collector](https://www.slideshare.net/GurpreetSachdeva2/java-garbage-collectors-moving-to-java7-garbage-first-g1-collector)
- [Detailed G1 Garbage Collector, programmersought](https://www.programmersought.com/article/87252966380/)
