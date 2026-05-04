Trong team mình, anti-pattern dễ xảy ra nhất là “đổ mọi thứ vào một bảng lớn rồi để query tự xử lý”, thay vì quản trị lifecycle Bronze–Silver–Gold một cách kỷ luật.

Lý do là vì cách làm này cho cảm giác “nhanh để ship”: ingest xong là có dữ liệu ngay, không cần định nghĩa rule làm sạch, dedup, hay metric contract. Nhưng khi dữ liệu tăng, chi phí đọc toàn bảng tăng mạnh, dashboard chậm, và tranh cãi về “single source of truth” bắt đầu xuất hiện vì mỗi người tự viết logic xử lý khác nhau.

Sau lab này, mình thấy rõ hậu quả kỹ thuật: thiếu compaction/Z-order gây file-skipping kém; thiếu time-travel discipline làm rollback rủi ro; thiếu Silver validation khiến Gold metric dễ sai. Vì vậy, nhóm mình nên chốt ngay data contract theo từng layer, lịch optimize định kỳ, và checklist kiểm chứng (row-count drop ở Silver, quality assertions, history/audit) trước khi publish Gold cho stakeholder.
