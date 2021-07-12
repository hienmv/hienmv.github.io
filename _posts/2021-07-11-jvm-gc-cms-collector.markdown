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

Black nodes points only black or gray nodes.

If gray set is empty — all nodes in white set can be garbaged.


---
## 2. CMS (todo)


---
**Reference**
- Optimize Java, Chapter 7: Advance Garbage collection
- [CMS collector, Oracale](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)
- [Java Garbage Collectors – Moving to Java7 Garbage First (G1) Collector](https://www.slideshare.net/GurpreetSachdeva2/java-garbage-collectors-moving-to-java7-garbage-first-g1-collector)
- [CMS, plumbr.io](https://plumbr.io/handbook/garbage-collection-algorithms-implementations/concurrent-mark-and-sweep)
- [GC, adamansky](https://adamansky.bitbucket.io/slides/gc/index.html)
