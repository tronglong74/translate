

**Blink-hash: Một chỉ mục kết hợp thích ứng cho cơ sở dữ liệu chuỗi thời gian lưu trữ trong bộ nhớ**

---

### Tóm tắt

Việc thu nhận dữ liệu tốc độ cao là rất quan trọng đối với các tải công việc chuỗi thời gian, được thúc đẩy bởi sự phát triển của các ứng dụng Internet of Things (IoT). Chúng tôi nhận thấy rằng các chỉ mục dựa trên cây truyền thống gặp phải vấn đề tắc nghẽn nghiêm trọng về khả năng mở rộng khi xử lý các tải công việc chuỗi thời gian chèn khóa dấu thời gian tăng dần đơn điệu vào một chỉ mục; tất cả các thao tác chèn đều xảy ra trong một vùng bộ nhớ nhỏ dẫn đến tranh chấp rất cao.

Trong nghiên cứu này, chúng tôi giới thiệu một thiết kế chỉ mục mới, **Blink-hash**, kết hợp chỉ mục dựa trên cây với các nút lá băm để giảm tranh chấp trong các thao tác chèn đơn điệu. Các thao tác chèn sẽ được phân phối ngẫu nhiên trong một nút băm (lớn hơn nhiều so với nút cây B+ truyền thống) nhằm giảm xung đột. Chúng tôi cũng phát triển các tối ưu hóa khác (xấp xỉ trung bình và tách lười) để tăng tốc độ tách nút băm. Ngoài ra, chúng tôi phát triển các tối ưu hóa thích ứng cấu trúc để chuyển đổi động một nút băm thành các nút cây B+ nhằm cải thiện hiệu suất quét.

Kết quả đánh giá của chúng tôi cho thấy **Blink-hash** đạt được thông lượng cao hơn lên đến 91,3 lần so với các chỉ mục truyền thống trong khối lượng công việc chuỗi thời gian với các thao tác chèn đơn điệu dấu thời gian vào chỉ mục, đồng thời có hiệu suất quét tương đương với cây B+ tối ưu.

---

### 1. Giới thiệu

Việc thu nhận dữ liệu tốc độ cao và xử lý truy vấn đang ngày càng trở nên quan trọng do sự bùng nổ của các thiết bị Internet of Things (IoT) như cảm biến, đầu đọc RFID, thiết bị chăm sóc sức khỏe, v.v. Trong các ứng dụng này, dữ liệu được đánh dấu thời gian liên tục và nhanh chóng được tạo ra với tốc độ hàng triệu sự kiện mỗi giây. Khối lượng lớn dữ liệu này không chỉ đòi hỏi một giải pháp lưu trữ dữ liệu hiệu quả mà còn cần khả năng mở rộng hiệu suất tuyến tính để tốc độ xử lý dữ liệu có thể theo kịp tốc độ tạo dữ liệu.

Trong khi nhiều khía cạnh của hệ thống cơ sở dữ liệu truyền thống đã được thiết kế lại để đáp ứng yêu cầu của việc thu nhận dữ liệu tốc độ cao và xử lý truy vấn hiệu quả, các chỉ mục lại chưa nhận được sự chú ý đủ lớn. Mặc dù có nhiều kỹ thuật đã được đề xuất để cải thiện hiệu suất chỉ mục về hiệu quả bộ nhớ đệm, hiệu quả không gian và kiểm soát đồng thời, các giải pháp hiện tại không nhắm đến những thách thức mới xuất hiện trong các tải công việc chuỗi thời gian.

Các đặc điểm của tải công việc chuỗi thời gian khác biệt so với các tải công việc OLTP thông thường. Đặc biệt, các khóa thường là dấu thời gian tăng dần đơn điệu. Đặc điểm này làm cho các chỉ mục dựa trên cây hiện tại trở nên không tối ưu trong các tải công việc chuỗi thời gian. Khi sử dụng dấu thời gian làm khóa chỉ mục, các thao tác chèn sẽ có xu hướng tập trung cao vào các nút lá phía phải nhất của cây, gây ra tranh chấp cực kỳ cao và tạo ra nút cổ chai về khả năng mở rộng.

---

### 2. Bối cảnh và động lực

Các tải công việc chuỗi thời gian đã trở thành một trong những tải công việc cốt lõi trong các hệ thống cơ sở dữ liệu. Các tải công việc này thường tạo ra một lượng lớn dữ liệu theo thời gian thực từ các thiết bị phân tán về mặt địa lý. Điều này yêu cầu khả năng thu nhận dữ liệu tốc độ cao và xử lý truy vấn hiệu quả, trong khi các chỉ mục truyền thống lại không mang lại hiệu suất tối ưu.

Hiện nay, hầu hết các hệ thống chuỗi thời gian vẫn sử dụng các chỉ mục B+-tree hoặc skip-lists như là các chỉ mục của chúng. Trong phần này, chúng tôi điều tra khả năng mở rộng của các chỉ mục tiêu biểu trên một bộ xử lý đa lõi bằng cách sử dụng tải công việc chèn dữ liệu mới với các khóa tăng dần đơn điệu (tức là, các dấu thời gian).

Hình 1 so sánh thông lượng của các chỉ mục cây và chỉ mục băm trong hai tải công việc khác nhau (chèn ngẫu nhiên và chèn đơn điệu) với số lượng luồng thay đổi. Chèn ngẫu nhiên chèn các khóa ngẫu nhiên phân phối đều trong khi chèn đơn điệu chèn các khóa tăng dần đơn điệu.

Các chỉ mục dựa trên cây không thể mở rộng trong các tải công việc chuỗi thời gian, nhưng chúng rất quan trọng vì hỗ trợ các quét phạm vi. Chúng tôi quan sát thấy rằng vấn đề này có thể được giải quyết bằng cách kết hợp cây B+ và băm vào một cấu trúc chỉ mục duy nhất để có được lợi ích của cả hai loại.

---

### 3. Thiết kế cơ bản của Blink-hash

**Blink-hash** kết hợp phương pháp băm và cây B+ thành một chỉ mục duy nhất. Một thách thức chính là kết hợp hai phương pháp này sao cho không làm giảm hiệu suất hoặc ảnh hưởng đến tính đúng đắn, đồng thời giữ cho thiết kế đơn giản. Đối với **Blink-hash**, chúng tôi quyết định giữ nguyên cấu trúc của **Blink-tree** càng nhiều càng tốt và chỉ thay thế các nút lá có lưu lượng chèn cao bằng các nút băm để giảm xung đột gây ra bởi các thao tác chèn đơn điệu.

**3.1 Cấu trúc dữ liệu**

Hình 3 mô tả cấu trúc tổng thể của **Blink-hash**. Một nút lá trong **Blink-hash** có thể là một nút cây B+ hoặc một nút băm. Một nút cây B+ chứa một con trỏ đến nút anh em bên phải của nó (giống như trong **Blink-tree**), trong khi một nút băm có các con trỏ đến cả nút anh em bên trái và bên phải của nó. Ban đầu, một nút lá bắt đầu như một nút băm và sẽ được chuyển đổi thành các nút cây B+ khi nó gặp phải các truy vấn quét.

**3.2 Các thao tác cơ bản trong Blink-hash**

**Đọc**: Người đọc đầu tiên tìm chỉ số bucket bằng cách sử dụng hàm băm và đọc giá trị của khóa tương ứng trong bucket.  
**Chèn**: Một người viết không cần khóa toàn bộ nút băm, chỉ cần khóa bucket chứa khóa chèn vào.  
**Cập nhật và Xóa**: Các thao tác cập nhật và xóa hoạt động tương tự như chèn, nhưng thay vì thêm một cặp khóa-giá trị mới, nó thay đổi hoặc xóa giá trị cho khóa tương ứng.  
**Quét**: Một quét phạm vi sẽ đọc các giá trị trong tất cả các bucket và sắp xếp chúng để trả về kết quả.  

---

### 4. Các tối ưu hóa

**4.1 Xấp xỉ trung bình**

Trong một thao tác tách nút băm, khóa trung bình được tìm bằng cách quét tất cả các bucket và sắp xếp các khóa thu thập được. Điều này gây ra chi phí truy cập bộ nhớ và sắp xếp đáng kể trong đường dẫn chính. Chúng tôi đưa ra nhận định rằng một khóa trung bình chính xác không cần thiết để đảm bảo tính đúng đắn của việc tách. Thay vào đó, một khóa gần với trung bình là đủ để đạt được hiệu suất chấp nhận được. Chúng tôi xác định một khóa gần đúng này thông qua việc lấy mẫu một tập con các khóa, thay vì đọc tất cả các khóa.

**4.2 Tách lười**

Việc di chuyển các cặp khóa-giá trị đến nút mới là một nút cổ chai khác về khả năng mở rộng trong thao tác tách nút băm. Chúng tôi giới thiệu kỹ thuật **tách lười** để giảm thiểu chi phí di chuyển bằng cách chia nhỏ phần tốn thời gian xử lý trên đường dẫn chính thành các phần nhỏ hơn tại mức độ bucket. Điều này giúp tránh việc di chuyển dữ liệu tốn kém

 trong đường dẫn chính và di chuyển dữ liệu một cách lười biếng.

---

### 5. Đánh giá

**5.1 Thiết lập thực nghiệm**

Chúng tôi đã triển khai **Blink-hash** dựa trên **Blink-tree** và sử dụng các lệnh SIMD để tăng tốc các thao tác trên các dấu vân tay (fingerprints) trong các nút băm. Tất cả các thí nghiệm được thực hiện trên một máy chủ CloudLab.

**5.2 Hiệu suất đơn luồng trong khối lượng công việc chuỗi thời gian**

Chúng tôi đầu tiên đánh giá hiệu suất đơn luồng của các chỉ mục trong khối lượng công việc chuỗi thời gian. **Blink-hash** cho thấy khả năng mở rộng tốt hơn nhiều so với các chỉ mục khác, đặc biệt là khi số luồng tăng lên.

**5.3 Khả năng mở rộng trong khối lượng công việc chuỗi thời gian**

Chúng tôi đánh giá khả năng mở rộng của các chỉ mục khi tăng số lượng luồng. **Blink-hash** là chỉ mục duy nhất có khả năng mở rộng tuyến tính trong các thao tác chèn đơn điệu, đạt được hiệu suất cao hơn đến 91,3 lần so với các chỉ mục khác khi sử dụng 32 luồng.

---

### 6. Kết luận

Chúng tôi đã giới thiệu **Blink-hash**, một chỉ mục kết hợp thích ứng cho cơ sở dữ liệu chuỗi thời gian lưu trữ trong bộ nhớ, kết hợp ưu điểm của cả cây B+ và chỉ mục băm. **Blink-hash** vượt trội hơn các chỉ mục truyền thống trong các thao tác chèn đơn điệu và quét phạm vi. Với khả năng mở rộng tuyến tính, **Blink-hash** mang lại hiệu suất cao hơn đến 91,3 lần so với các chỉ mục khác trong tải công việc chuỗi thời gian.

---
