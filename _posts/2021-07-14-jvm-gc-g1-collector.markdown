---
layout: post
title: "[JVM][GC] Garbage-First collector"
date: 2021-07-14 22:47:52 +0900
categories: tech

---
>Bài viết gồm 4 phần:
>- G1 Heap Layout and Regions.
>- G1 Algorithms Design.
>- Young Generation Collection with G1
>- Old Generation Collection with G1

---
Garbage-First (G1) collector là GC collector khác rất nhiều so với CMS hay parallel collector, được thiết kế hướng tới những thiết bị multi-processor với lượng memory lớn, và đáp ứng được các yêu cầu như: (GC) pause time goals with a high probability; high throughput.

G1 collector là default collector từ Java 9, thay thế cho parallel collectors.

G1 được thiết kế cho các ứng dụng yêu cầu bộ nhớ heap lớn với GC latency giới hạn. Với Heap sizes khoảng 6GB hoặc hợn, thời gian pause time ổn định và dự doán được dưới 0.5s. 

Những ứng dụng đang dùng CMS hoặc ParallelOld GC mà có một hoặc nhiều đặc điểm dưới đây sẽ rất lợi nếu chuyển sang dùng G1: 
- Full GC duration quá lâu hoặc liên tục xảy ra.
- Tỉ lệ cấp phát bộ nhớ cho các object hoặc việc move object sang vùng old generation quá lớn. 
- Thời gian collection pause hoặc compaction pause dài: (lớn hơn 0.5s tới 1s).

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
- Mặc định kích thước của 1 region này là 1MB, (sẽ lớn hơn đối với heap có kích thước lớn hơn). G1 chấp nhận các region có kích thước là 1, 2, 4, 8, 16, 32, 64 MB.
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
  - Khi khởi động, G1 sẽ thống kê có bao nhiêu regions mà có thể thu thập ở mỗi GC cycle. Nếu lượng memory được thu thập cân bằng với số lượng object mới cần cấp phát kể từ lần GC cycle trước đó, thì với những object mới này, G1 sẽ không cần cấp phát thêm vùng nhớ mới nữa, mà sẽ dùng lại vùng nhớ vừa mới được thu thập. 


**Recal: Weak Generational Hypothesis**
1. Sự phân phối vòng đời của objects (object lifetime) ở JVM: phần lớn các object là short-lived objects (young objects) và phần còn lại là là long-lived object. (old objects)
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
## 3. Young generation Collection with G1

Summary: 
- Heap sẽ được chia nhỏ thành các regions.
- Young generation memory là một danh sách (set) của các regions không liên tục với nhau. 
- Young GC được thực hiện song song bằng cách sử dụng multithreads. Khi Young generation GCs xảy ra, toàn bộ các application threads sẽ bị dừng (STW), và các live object sẽ được copy tới regions suvivor hoặc old generation.  

Cụ thể:
### Young Generation in G1
Heap sẽ được chia thành khoảng 2000 regions, với size từ 1MB tới 32MB, và các regions thì không nhất cần phải liên tục như ở Parallel và CMS collector.

![](../assets/g1-young-generation.png)

### A young GC in G1
Live objects sẽ được copy/move tới 1 hoặc nhiều survivor regions khác. Nếu đạt ngưỡng tồn tại đủ lâu (aging threshold), một số object sẽ được move qua old generation regions.

![](../assets/g1-young-gc-1.png)

STW diễn ra. Eden size and survivor size sẽ được tính toán cho lần young GC tiếp theo. Thông tin thống kê sẽ được lưu lại để giúp cho việc tính toán size của các region.

### End of a young GC with G1

Live object sẽ được copy/move tới survivor hoặc old generation regions. 

![](../assets/g1-young-gc-2.png)


## 4. Old Generation Collection with G1

Summary:
- Concurrent Marking Phase
    - Liveness information is calculated concurrently while the application is running.
    - This liveness information identifies which regions will be best to reclaim during an evacuation pause.
    - There is no sweeping phase like in CMS.
- Remark Phase
    - Uses the Snapshot-at-the-Beginning (SATB) algorithm which is much faster then what was used with CMS.
    - Completely empty regions are reclaimed.
- Copying/Cleanup Phase
    - Young generation and old generation are reclaimed at the same time.
    - Old generation regions are selected based on their liveness.

Gồm 5 phase:

1. **Initial Mark** (STW)
Đánh dấu các survivor regions (root regions) mà có thể có reference tới các objects ở old generation. 
2. **Root Region Scanning**
Scan các survivor regions mà có reference tới old generation. Phase này xảy ra đồng thời khi application threads đang chạy, và phải hoàn thành trước khi một young GC có thể xảy ra. 
3. **Concurrent Marking**
Tìm các live object trên toàn bộ heap. Phase này xảy ra đồng thời khi application threads đang chạy, và nó có thể bị gián đoạn bở young GC. 
4. **Remark** (STW)
Đánh dấu toàn bộ live object trên heap. 
Sử dụng thuật toán SATB (snapshot-at-the-beginning) nhanh hơn nhiều so với thuật toán được dùng ở CMS collector. 
5. **Cleanup** (STW and Concurrent)	
- Thực hiện ở các live objects và các regions hoàn toàn free. (STW)
- Scrubs the Remembered Sets. (Stop the world)
- Reset các empty regions và cho chúng vào free list (concurrent).
- Copying (STW): Copy/move object live object tới các regions chưa được sử dụng. Điều này có thể được thực hiện với các young generation regions (được gọi là GC pause (young)), hoặc với đồng thời cả young và  old generation regions (được gọi là GC pause (mixed)).

### Initial Marking Phase

Đánh dấu các live objects sau khi young generation garbage collection.

![](../assets/g1-old-gc-1.png)

### Concurrent Marking Phase

Nếu tìm thấy những empty regions (được đánh dấu bởi "X" ở hình dưới), thì chúng sẽ được xoá ngay lập tức ở `remark phase`, và thông tin thống kê (`accounting information`) (xác định thông tin về live objects (`liveness`)) sẽ được tính toán. 

![](../assets/g1-old-gc-2.png)


### Remark Phase

Những empty regions sẽ được xoá và lấy lại bộ nhớ. Thông tin `Region liveness` sẽ được tính toán cho toàn bộ các regions. 

![](../assets/g1-old-gc-3.png)

### Copying/Cleanup Phase

G1 sẽ lựa chọn các regions mà có "liveness" thấp nhất, mà các regions có thể được collect nhanh nhất. Những regions này sẽ được collect đồng thời ở thời điểm của young GC. Tức là, cả young và old generation sẽ được collect đồng thời.

![](../assets/g1-old-gc-4.png)

### After Copying/Cleanup Phase

Những regions được lựa chọn ở trên để collect và compact vào dark blue region phía dưới.

![](../assets/g1-old-gc-5.png)

---
**Reference**
- Optimize Java, Chapter 7: Advance Garbage collection
- [G1 collector, Oracale](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)
- [Java Garbage Collectors – Moving to Java7 Garbage First (G1) Collector](https://www.slideshare.net/GurpreetSachdeva2/java-garbage-collectors-moving-to-java7-garbage-first-g1-collector)
- [Detailed G1 Garbage Collector, programmersought](https://www.programmersought.com/article/87252966380/)
