---
layout: post
title: "[JVM][GC] Concurrent Mark and Sweep (CMS) collector"
date: 2021-07-11 22:47:52 +0900
categories: tech

---
Bài viết gồm 2 phần:
- Concurrent GC Theory.
- CMS.

## 1. Concurrent GC Theory

### JVM Safepoints
Để thực hiện `fully STW garbage collection` (ví dụ như khi sử dụng HotSpot parallel collectors), toàn bộ application threads sẽ bị tạm ngừng cho đến khi GC cycle kết thúc. Không có cách nào để một GC thread đưa ra yêu cầu cho OS để buộc một application thread ngừng, vậy nên những application threads (là một phần của JVM process) mà đang thực thi cần phải phối hợp với nhau để đạt được điều này. 

JVM yêu cầu mỗi application thread có `safepoints` - nơi mà dữ liệu bên trong mỗi thread ở một trạng thái "tốt" (known-good state) và khi một thread ở trạng thái này, nó có thể ngừng lại để làm các hành động phối hợp (`coordinated actions`), ví dụ như STW,..

Khi cài đặt JVM, có 2 quy tắc chính chi phối cách tiếp cận của JVM tới `safepoint`: 
- JVM không thể bắt buộc 1 thread vào trạng thái `safepoint state`.
- JVM không thể ngăn 1 thread "rời" khỏi trạng thái `safepoint state`.

Một trường hợp tổng quát khi đạt trạng thái safepoint:
1. JVM sử dụng 1 global flag gọi là `time to safepoint flag`.
2. Mỗi application thread sẽ check và xác nhận xem flag trên đã được set hay chưa.
3. Khi flag trên đã được set (`= true`), application threads sẽ tạm ngừng và chờ để được "đánh thức". 

Một số case đặc biệt khác:
- Một thread sẽ được tự động ở trạng thái safepoint nếu nó:
    - bị block trên một monitor.
    - đang thực thi JNI code.
- Một thread không cần ở trang thái safepoint nếu nó:
    - là một phần của việc thực thực bycode (interpreted mode).
    - bị gián đoạn bởi OS.

### Tri-Color Marking
- Maintain 3 objects sets.
    - White sets: unknown status/ not yet visited.
    - Gray sets: Live, children not known/ visited.
    - Black sets: Live, children live too / visited + child visited.
- Initially all objects are white
- Grey all GC roots
- While gray set is not empty
    - If it has white child → color child grey
    - Otherwise color node black
    - Go to the step 3

Black nodes points only black or gray nodes. All black objects have been proven to be reachable and must remain alive.

If gray set is empty — all nodes in white set can be garbaged since white nodes are eligible for collection and correspond to objects that are no longer reachable.

![](../assets/tri-color-marking.png)


---
## 2. CMS

CMS collector được thiết kế để giảm GC pause-time ở vùng nhớ Tenured (aka old generation), và nó thường được sử dụng kèm với `ParNew` - một biến thể của `parallel collector` ở vùng nhớ `young generation`.

**Ý tưởng**: CMS sẽ thực thi nhiều nhất có thể, trong khi các application thread vẫn đang chạy nhằm giảm thiểu GC pause time nhiều nhất có thể.
Thuật toán marking được sử dụng là `tri-color marking`. Vì một số step của CMS, GC thread chạy đồng thời với application thread, nên object graph có thể bị chỉnh sửa trong qua trình collector scan vùng nhớ heap. Vì vậy, CMS phải có 1 step để chỉnh sửa lại kêt quả của nó (remark) nhằm không vi phạm rule thứ 2 của một garbag collector: không thu thập bất kỳ live object nào.

Các phases của CMS phức tạp hơn nhiều ở parallel collectors: 
1/ Initial Mark (STW)
2/ Concurrent Mark
3/ Concurrent Preclean
4/ Remark (STW)
5/ Concurrent Sweep
6/ Concurrent Reset

Hầu hết các phases, GC sẽ chạy đồng hành cùng với các application threads. Tuy nhiên, ở 2 phase (`Initial Mark` và `Remark`), toàn bộ application threads sẽ bị ngừng, điều này làm giảm GC pause time đáng kể s với 1 `long STW` (như ở parallel collector)

**Phase 1: Initial Mark** đánh dấu toàn bộ cac object ở vùng nhớ `old generation` để xác định xem nó được liên kết trực tiếp từ GC roots hay là được tham chiếu từ một vài live object ở vùng nhớ Young generation. Việc xác định này là quan trọng vì Old generation sẽ được xử lý riêng biệt so với Young generation. Điều này sẽ cho phép `marking phase` sẽ tập trung vào 1 GC pool mà không cần quan tâm tơi vùng nhớ khác.

![](../assets/phase-1-init-mark-cms.png)

**Phase 2: Concurrent Mark** sử dụng tri-color marking trên heap và lưu trữ các thay đổi để có thể sử lý sau. Bước này sẽ chạy đồng thời với application mà không làm ngưng bất kỳ application threads nào. Một chú ý là không phải tất cả các live object ở vùng nhơ Old generation sẽ được đánh dấu, vì application thread vẫn đang chạy trong khi marking.

![](../assets/phase-2-concurrent-mark-cms.png)

**Phase 3: Concurrent Preclean** bước này chạy song song với các application threads, và có nhiệm vụ giảm thời gian STW ở remark pase nhiều nhất có thể. 
While the previous phase was running concurrently with the application, some references were changed. Whenever that happens, the JVM marks the area of the heap (called “Card”) that contains the mutated object as “dirty” (this is known as Card Marking).

The Concurrent Preclean phase appears to try to reduce the length of the STW Remark phase as much as possible. The Remark phase uses the card tables to fix up the marking that might have been affected by mutator threads during the Concurrent Mark phase.

Một số ảnh hưởng khi sử dụng CMS: (không phải ảnh hưởng nào cũng là tích cực).
The observable effects of using CMS are as follows, for most workloads:
- Application threads don’t stop for as long
- A single full GC cycle takes longer (in wallclock time) 
- Application throughput is reduced while a CMS GC cycle is running
- GC uses more memory for keeping track of objects.
- Considerably more CPU time is needed overall to perform GC.
- CMS does not compact the heap, so Tenured can become fragmented.

CMS collector hiện từ bản Java 9 trở đi được đánh giá là `deprecated`, và tới Java 14 thì ngừng hỗ trợ hoàn toàn.

---
**Reference**
- Optimize Java, Chapter 7: Advance Garbage collection
- [CMS collector, Oracale](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)
- [Java Garbage Collectors – Moving to Java7 Garbage First (G1) Collector](https://www.slideshare.net/GurpreetSachdeva2/java-garbage-collectors-moving-to-java7-garbage-first-g1-collector)
- [CMS, plumbr.io](https://plumbr.io/handbook/garbage-collection-algorithms-implementations/concurrent-mark-and-sweep)
- [GC, adamansky](https://adamansky.bitbucket.io/slides/gc/index.html)
