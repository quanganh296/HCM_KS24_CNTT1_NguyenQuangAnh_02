Câu 1:
1. Lỗ hổng thao tác giá từ phía Client:

-Vị trí lỗi: Dòng order.setTotalPrice(request.getUnitPrice() * request.getQuantity());
lấy trực tiếp đơn giá (unitPrice) từ orderRequestDTO do người dùng gửi lên thay vì truy xuất từ cơ sở dữ liệu

-Hậu quả: Kẻ tấn công có thể can thiệp vào gói tin HTTP request, sửa unitPrice thành 0,1 hoặc một số âm. Hậu quả là
khách hàng mua được hàng giá trị cao với giá thấp, gây thâất thoát tài chính cho doanh nghiệp.

2. Lỗi tranh chấp dữ liệu đồng thời:
- Vị trí lỗi: if(products.getStock() >= request.getQuantity()); và cập nhật tồn khi product.setStock(....) được
thực hiện tuàn tự trên bộ nhớ ứng dụng mà không có cơ chế khóa hay xử lý đồng bộ dưới Database.

-Hậu quả: Xảy ra tình trạng bán vượt quá số lượng tồn kho thực tế hoặc âm kho, dẫn đến không có hàng giao cho khách.

Câu 2: Giải pháp:

1. Khắc phục lỗ hổng thao túng giá:

-Xóa trường unitPrice khỏi lớp orderRequestDTO
-Lấy đơn giá chuẩn từ thực thể Product đã truy vấn trong Database để tỉnh tổng tiền:
order.setTotalPrice(product.getPrice() * request.getQuantity());


2. Khắc phục lỗi truy cập đồng thời:

-Thực hiện kiếm tra và trừ kho trực tiếp trong 1 câu lệnh UPDATE tại ProductRepository:
   @Modifying
   @Query("UPDATE Product p SET p.stock = p.stock - :quantity WHERE p.id = :id AND p.stock >= :quantity")
   int decreaseStock(@Param("id") Long id, @Param("quantity") int quantity);

Câu 3:
-Lỗi bộc lộ rõ nhất là lỗi: Tranh chấp dữ liệu khi truy cập đồng thời
-Giải thích: Trong các đợt Flash Sale, hàng ngàn người dùng sẽ bấm đặt mua cùng một sản phẩm có số lượng giới hạn tại
cùng một mili-giây. Do hàm placeOrder thực hiện đọc tồn kho (getStock) và ghi tồn kho (save()) tách rời mà không có
cơ chế khóa (Lock), hàng chục luồng xử lý đồng thời sẽ cùng đọc được trạng thái sản phẩm "còn hàng" trước khhi luồng đầu
tiên kịp cập nhật số lượng mới xuống Database. Kết quả là hệ thống cho phép tạo thành công nhiều đơn hàng hơn số lượng
hàng thực tế trong kho và làm số lượng tồn kho bị sai lệch hoàn toàn