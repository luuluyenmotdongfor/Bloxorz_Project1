# Bloxorz Solver

Đồ án 1 môn **Cơ sở Trí tuệ nhân tạo (CSC14003)**, Trường Đại học Khoa học Tự nhiên, ĐHQG-HCM.

Dự án mô hình hóa trò chơi Bloxorz thành bài toán tìm kiếm trong không gian trạng thái và cài đặt năm thuật toán bằng Python: **BFS**, **DFS**, **IDS**, **UCS** và **A\***. Bộ màn chơi gồm **20 bản đồ**, có nền thường, gạch dễ vỡ, cầu và các loại công tắc.

## 1. Bài toán Bloxorz

Người chơi điều khiển khối hộp kích thước `1 × 1 × 2` lăn trên bàn cờ. Mục tiêu là đưa khối nguyên vẹn đến ô đích ở tư thế đứng thẳng.

Các cơ chế được xử lý trong phần lõi:

- **Di chuyển:** Khối lăn theo bốn hướng lên, xuống, trái và phải. Nước đi ra ngoài bản đồ hoặc vào ô không có nền đỡ bị loại.
- **Gạch dễ vỡ:** Không cho phép khối nguyên vẹn đứng thẳng; khối nằm hoặc mảnh đơn sau khi tách có thể đi qua.
- **Cầu:** Chỉ đi qua được khi cầu đang mở. Trạng thái đóng/mở của cầu được lưu trong trạng thái tìm kiếm.
- **Soft switch:** Kích hoạt khi một phần của khối đi vào công tắc.
- **Heavy switch:** Chỉ kích hoạt khi khối nguyên vẹn đứng thẳng trên công tắc.
- **Split switch:** Tách khối đang đứng thành hai mảnh đơn và đưa chúng tới hai vị trí xác định. Mỗi lần chỉ điều khiển một mảnh; hai mảnh tự ghép khi di chuyển tới vị trí kề cạnh nhau.

Công tắc điều khiển cầu hỗ trợ hai chế độ: `toggle` để đảo trạng thái cầu và `permanent` để đặt cầu về trạng thái cố định.

## 2. Mô hình hóa trạng thái

Lớp `State` lưu tọa độ, tư thế của khối và trạng thái các cầu. Khi khối bị tách, trạng thái lưu thêm tọa độ mảnh thứ hai và mảnh đang được điều khiển.

| Tư thế | Các ô khối chiếm chỗ |
| --- | --- |
| `STANDING` | Một ô `(r, c)` |
| `LYING_H` | Hai ô `(r, c)` và `(r, c + 1)` |
| `LYING_V` | Hai ô `(r, c)` và `(r + 1, c)` |
| `SPLIT` | Hai ô `(r, c)` và `(r2, c2)` |

Cùng vị trí nhưng khác tư thế, trạng thái cầu hoặc mảnh đang điều khiển được xem là các trạng thái khác nhau. Các phương thức so sánh và băm của `State` hỗ trợ nhận diện trạng thái lặp trong quá trình tìm kiếm.

Khối nguyên vẹn có bốn hành động: `UP`, `DOWN`, `LEFT`, `RIGHT`. Khi ở trạng thái `SPLIT`, có thêm `SWITCH_CUBE` để đổi mảnh điều khiển, nên có tối đa năm hành động ứng viên. Mỗi hành động, kể cả đổi mảnh, được tính là một bước của lời giải.

Lớp `Board` đọc bản đồ JSON, sinh các trạng thái kế tiếp hợp lệ, xử lý công tắc và kiểm tra đích. Trạng thái đích yêu cầu khối nguyên vẹn đứng thẳng tại ô đích; một mảnh đơn chưa thể kết thúc màn chơi.

## 3. Các thuật toán tìm kiếm

| Thuật toán | Cách tìm kiếm | Đặc điểm lời giải |
| --- | --- | --- |
| **BFS** | Hàng đợi FIFO, duyệt theo độ sâu tăng dần | Tối ưu số hành động |
| **DFS** | Ngăn xếp LIFO, duyệt sâu trước | Không bảo đảm ít hành động nhất hoặc chi phí thấp nhất |
| **IDS** | Lặp tìm kiếm giới hạn độ sâu từ 0 đến 100 | Tối ưu số hành động khi tìm được lời giải trong các giới hạn đã đặt |
| **UCS** | Hàng đợi ưu tiên theo chi phí tích lũy `g` | Tối ưu tổng chi phí theo hàm cost của dự án |
| **A\*** | Hàng đợi ưu tiên theo `f = g + h` | Dùng heuristic để hướng dẫn tìm kiếm, cùng mục tiêu tối ưu chi phí với UCS |

Các solver kế thừa `BaseSolver`, dùng chung mô hình chuyển của `Board` và trả về chuỗi hành động khi tìm được lời giải. Nếu trạng thái đầu đã là đích, lời giải là danh sách rỗng.

Kết quả phân biệt các trạng thái `OK`, `UNSOLVABLE`, `TIME_LIMIT`, `NODE_LIMIT` và `DEPTH_LIMIT`. Dừng do giới hạn thời gian, số lượt mở rộng hoặc độ sâu không đồng nghĩa với vô nghiệm.

## 4. Hàm chi phí và heuristic

UCS và A* cùng sử dụng `get_step_cost(board, next_state)` trong `Source/solvers/search_utils.py`. Chi phí mỗi hành động được tính trên trạng thái sau hành động:

`cost = 10 + 20 × F + 20 × S + B`

Trong đó:

- `F = 1` nếu có ít nhất một ô khối đang chiếm là gạch dễ vỡ, ngược lại bằng 0.
- `S = 1` nếu trạng thái sau hành động là `SPLIT`, ngược lại bằng 0.
- `B` là số cầu đang đóng, tính theo số định danh cầu, không phải số ô cầu.

Chi phí bước đi luôn dương. Vì các bước có thể có chi phí khác nhau, lời giải ít hành động nhất chưa chắc có tổng chi phí thấp nhất.

Heuristic của A* được xây dựng bằng Dijkstra chạy ngược từ ô đích trên đồ thị nới lỏng. Đồ thị bao gồm toàn bộ lưới chữ nhật, kể cả các ô trống, và bỏ qua ràng buộc về tư thế khối, cầu và gạch dễ vỡ. Cạnh giữa hai ô kề nhau có trọng số 5; cạnh dịch chuyển từ công tắc Split tới điểm đến có trọng số 0.

Giá trị `h` của một trạng thái là khoảng cách nới lỏng nhỏ nhất trong các ô mà khối đang chiếm. Bảng khoảng cách được tính khi khởi tạo solver và dùng lại trong quá trình tìm kiếm.

## 5. Cấu trúc phần lõi

| Đường dẫn | Chức năng |
| --- | --- |
| `Source/core/state.py` | Biểu diễn trạng thái, tư thế, các ô chiếm chỗ, so sánh và băm trạng thái |
| `Source/core/board.py` | Nạp bản đồ, xử lý luật chơi, sinh nước đi hợp lệ và kiểm tra đích |
| `Source/solvers/base_solver.py` | Giao diện chung, giới hạn tìm kiếm và thông tin kết quả của solver |
| `Source/solvers/bfs_solver.py` | Cài đặt BFS |
| `Source/solvers/dfs_solver.py` | Cài đặt DFS |
| `Source/solvers/ids_solver.py` | Cài đặt IDS và tìm kiếm giới hạn độ sâu |
| `Source/solvers/ucs_solver.py` | Cài đặt UCS |
| `Source/solvers/astar_solver.py` | Cài đặt A* |
| `Source/solvers/search_utils.py` | Hàm chi phí dùng chung và heuristic |
| `Source/levels/` | Dữ liệu JSON của 20 màn chơi |
