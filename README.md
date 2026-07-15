# LVTN — Nhật ký tiến độ đồ án tốt nghiệp

Kho lưu trữ báo cáo tiến độ hàng tuần, biên bản họp với giáo viên hướng dẫn, và các tài liệu liên quan đến đồ án tốt nghiệp.

**Tên đề tài:** Hệ thống tra cứu và liên kết văn bản pháp luật Việt Nam  
**Sinh viên thực hiện:**  
**Giáo viên hướng dẫn:**  
**Học kỳ:**  

---

## Cấu trúc thư mục

```
lvtn-progress/
├── weekly-reports/          # Báo cáo tiến độ hàng tuần
│   └── YYYY-WXX/            # Thư mục theo năm-tuần (ví dụ: 2025-W01)
│       └── report.md
├── meetings/                # Biên bản họp với GVHD
│   └── YYYY-MM-DD/          # Thư mục theo ngày họp
│       ├── agenda.md        # Chương trình cuộc họp
│       ├── notes.md         # Biên bản ghi chép
│       └── action-items.md  # Danh sách việc cần làm sau họp
├── deliverables/            # Các sản phẩm/milestone của đồ án
├── templates/               # Mẫu tài liệu
│   ├── weekly-report.md
│   ├── meeting-notes.md
│   └── action-items.md
├── resources/               # Tài liệu tham khảo, link hữu ích
│   └── references.md
└── progress-tracker.md      # Bảng theo dõi tiến độ tổng quan
```

---

## Quy ước đặt tên

| Loại tài liệu | Quy ước thư mục | Ví dụ |
|---|---|---|
| Báo cáo tuần | `weekly-reports/YYYY-WXX/` | `weekly-reports/2025-W03/` |
| Biên bản họp | `meetings/YYYY-MM-DD/` | `meetings/2025-01-15/` |

---

## Hướng dẫn sử dụng

1. **Tạo báo cáo tuần mới:** Copy `templates/weekly-report.md` vào thư mục `weekly-reports/YYYY-WXX/report.md`.
2. **Ghi biên bản họp:** Tạo thư mục `meetings/YYYY-MM-DD/`, copy template và điền nội dung.
3. **Cập nhật tiến độ tổng quan:** Cập nhật `progress-tracker.md` sau mỗi milestone.
