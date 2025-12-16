## Testing Plan - Product Image Upload

### Case chính

1. **Tạo sản phẩm không có ảnh**
   - Điền form đầy đủ, không upload ảnh.
   - Submit thành công, product tạo mới không có `imageFileId`.
2. **Tạo sản phẩm có ảnh**
   - Chọn file ảnh hợp lệ.
   - Upload lên ImageKit thành công, nhận `fileId`.
   - Preview ảnh hiển thị đúng trong form.
   - Submit form, product lưu `imageFileId` trong DB.
3. **Xem chi tiết sản phẩm có ảnh**
   - Mở `ProductDetailModal`.
   - Ảnh hiển thị đúng theo `imageFileId`.
4. **Cập nhật ảnh sản phẩm**
   - Mở form edit, upload ảnh mới.
   - `imageFileId` cập nhật, preview đổi theo ảnh mới.
5. **Lỗi upload**
   - Mô phỏng lỗi mạng / auth fail:
     - FE hiển thị thông báo lỗi thân thiện.
     - Không submit form nếu chưa có kết quả upload hợp lệ (nếu ảnh là bắt buộc).


