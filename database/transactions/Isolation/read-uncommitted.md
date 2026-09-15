Đúng: nếu xét về **data correctness**, `Read Uncommitted` gần như không giải quyết anomaly nào cả.

Trade-off của nó là:

> Hy sinh độ chính xác và tính nhất quán của kết quả đọc để giảm việc reader phải chờ writer.

## Nó giải quyết vấn đề nào?

Giả sử Transaction A đang cập nhật một bảng lớn và giữ lock trong 20 giây.

Với `Read Committed`:

```text
Transaction A đang ghi và giữ lock
                    ↓
Transaction B muốn đọc → có thể phải chờ
```

Với `Read Uncommitted`:

```text
Transaction A đang ghi và chưa commit
                    ↓
Transaction B vẫn đọc → không chờ dữ liệu được commit
```

Do đó, thứ mà `Read Uncommitted` tối ưu là:

| Đạt được                                   | Phải đánh đổi                        |
| ------------------------------------------ | ------------------------------------ |
| Ít reader–writer blocking                  | Có thể đọc dữ liệu chưa commit       |
| Latency đọc thấp hơn khi contention cao    | Kết quả giữa hai lần đọc có thể khác |
| Một số workload có throughput cao hơn      | Có thể thiếu hoặc thừa row           |
| Truy vấn vẫn chạy khi writer giữ data lock | Không có snapshot nhất quán          |

Nó giải quyết **contention/liveness**, không giải quyết **correctness**.

## “Cả ba anomaly có thể xảy ra” có nghĩa là gì?

Không có nghĩa mỗi query chắc chắn sai. Anomaly chỉ xảy ra khi có các transaction đồng thời xen kẽ đúng tình huống.

Nếu không ai đang cập nhật bảng, kết quả có thể hoàn toàn đúng.

```text
Không có concurrent write
→ Read Uncommitted có thể cho kết quả giống Read Committed
```

Nhưng database không đưa ra cam kết rằng kết quả ấy đáng tin:

```text
Có concurrent write
→ kết quả có thể đúng, cũ, chưa commit hoặc không nhất quán
```

## Khi nào sự đánh đổi này có thể chấp nhận?

Ví dụ một dashboard nội bộ hiển thị:

```text
Số request đang được xử lý: khoảng 10.000
```

Nếu kết quả thực tế là `10.012`, nhưng dashboard tạm thời hiển thị `10.008`, và vài giây sau tự refresh, sai lệch đó có thể không quan trọng.

Một số trường hợp hạn chế:

* Kiểm tra/debug dữ liệu nhanh.
* Dashboard gần đúng, tự refresh.
* Telemetry không dùng để ra quyết định nghiệp vụ.
* Truy vấn chẩn đoán mà việc chờ có hại hơn kết quả không chính xác.
* Dữ liệu mà người dùng chấp nhận chỉ mang tính tham khảo.

## Khi nào không nên dùng?

```sql
SELECT SUM(Balance) FROM Accounts;
SELECT Quantity FROM Inventory;
SELECT Status FROM Payments;
```

Không nên dùng nếu kết quả phục vụ:

* Thanh toán và số dư.
* Tồn kho.
* Báo cáo tài chính.
* Kiểm tra business rule.
* Ra quyết định cập nhật tiếp theo.
* Phân trang hoặc xuất báo cáo cần chính xác.

Ví dụ cực kỳ nguy hiểm:

```sql
IF (SELECT Quantity FROM Inventory) > 0
    -- Tạo đơn hàng
```

Nếu đọc phải giá trị chưa commit, hệ thống có thể tạo đơn dựa trên số lượng hàng không thực sự tồn tại.

## Tại sao ANSI vẫn định nghĩa nó?

Vì bảng ANSI mô tả một **phổ đảm bảo**:

```text
Ít isolation                                  Nhiều isolation
───────────────────────────────────────────────────────────→
Read Uncommitted → Read Committed → Repeatable Read → Serializable

Ít chờ hơn                                      Đúng đắn hơn
Concurrency cao hơn                             Chi phí cao hơn
```

`Read Uncommitted` chính là điểm thấp nhất của phổ: transaction tồn tại, nhưng việc đọc gần như không được bảo vệ khỏi ảnh hưởng của transaction khác.

## Trong hệ thống hiện đại

Trong đa số ứng dụng nghiệp vụ, mình sẽ không chọn `Read Uncommitted` chỉ để chữa blocking. Nên xem xét:

1. Tối ưu query và index.
2. Rút ngắn transaction.
3. Tránh giữ transaction khi gọi network/API.
4. Dùng `Read Committed` dựa trên MVCC/row versioning.
5. Dùng Snapshot Isolation nếu cần snapshot nhất quán.
6. Dùng read replica cho reporting.

Ví dụ SQL Server thường có lựa chọn tốt hơn:

```sql
ALTER DATABASE MyDatabase
SET READ_COMMITTED_SNAPSHOT ON;
```

Reader có thể đọc version đã commit thay vì chờ writer, nên đạt được mục tiêu **giảm blocking** mà không phải chấp nhận Dirty Read.

Kết luận:

> `Read Uncommitted` vẫn có công dụng, nhưng rất hẹp: khi “có kết quả ngay, dù chỉ gần đúng” quan trọng hơn “kết quả phải đáng tin”. Với business application thông thường, `Read Committed` + MVCC/row versioning thường là lựa chọn hợp lý hơn.

