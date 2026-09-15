Isolation là phần của ACID trả lời câu hỏi:
Khi nhiều transaction chạy cùng lúc, làm sao để chúng không nhìn 
thấy hoặc ghi đè lên những trạng thái trung gian gây sai dữ liệu?

anomalies của Isolation là những hiện tượng sai lệch có thể xảy ra 
khi nhiều transaction chạy đồng thời và nhìn/ghi dữ liệu của nhau
 theo cách không được kiểm soát đủ chặt.

| Anomaly             | Điều gì khien no bi sai?                |
| ------------------- | --------------------------------        |
| Dirty Read          | T1 upsert nhung chưa commit, T2 dirty   |
| Non-repeatable Read | T1 can read 1 row 2 lan, T2 upsert      |
|                     | giua 2 lan read T1=> T1 Non             |
| Phantom Read        | T1 can read 1 range 2 lan, T2 upsert    |
|                     | giua 2 lan read T1=> T1 Phantom         |
| Lost Update         | T1 va T2 deu upsert vao 1 resource =>   |
|                     | upsert T1 or T2 thay vi upsert T1 and T2|

Anomaly là hiện tượng lỗi/concurrency problem.
Isolation Level không phải “thuật toán giải anomaly”, mà là mức quy định/contract về anomaly nào được phép xảy ra và anomaly nào phải bị ngăn.
Còn Locking, MVCC, optimistic concurrency... mới gần với cơ chế/thuật toán thực thi để đạt isolation level đó.

Theo mô hình ANSI SQL kinh điển:

Isolation Level	    Dirty Read	Non-repeatable Read	Phantom Read
Read Uncommitted	Có thể	        Có thể	            Có thể
Read Committed	    Chặn	        Có thể	            Có thể
Repeatable Read	    Chặn	        Chặn	            Có thể
Serializable	    Chặn	        Chặn	            Chặn
Còn Lost Update không nằm gọn trong bảng ANSI cổ điển này; nó thường được xử lý bởi write locking, serializable execution, MVCC conflict detection, optimistic version check hoặc cách viết atomic update.
Correctness
    ▲
    │            Serializable
    │                 ●
    │
    │       Repeatable Read
    │            ●
    │
    │   Read Committed
    │        ●
    │
    │ Read Uncommitted
    │      ●
    └────────────────────────► Block Concurrency / Low Performance

Read Uncommitted: [Read Uncommitted](./read-uncommitted.md)
Read Uncommitted: [Read Uncommitted](./read-uncommitted.md)
Read Uncommitted: [Read Uncommitted](./read-uncommitted.md)
Read Uncommitted: [Read Uncommitted](./read-uncommitted.md)