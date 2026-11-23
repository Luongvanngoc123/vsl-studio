# Báo cáo quá trình xây dựng và cách làm của dự án VSL Studio

*Ngày viết: 23/11/2026. Báo cáo kể lại dự án được làm ra như thế nào từ 12/09/2026 đến 23/11/2026 và hệ thống hiện hoạt động theo cách nào. Nội dung soạn từ lịch sử git, nhật ký và tài liệu của dự án; nguồn của các con số ghi ở mục 9. Đây là báo cáo kỹ thuật, không phải chứng nhận độ chính xác trên người ký mới.*

## 1. Tóm tắt

VSL Studio (tên kho mã: VSL Virtual Lab) nhận dạng ngôn ngữ ký hiệu Việt Nam (VSL) từ camera: người ký làm ký hiệu trước camera, máy viết ra từ tiếng Việt tương ứng. Hệ thống gồm một máy chủ chạy trên máy tính cá nhân, một trang web, một ứng dụng Android và một lõi nhận dạng viết bằng Rust dùng chung cho máy tính và điện thoại.

1. **Hiện trạng (23/11/2026).** Hệ thống nhận 400 từ đơn của một mô hình nền đã huấn luyện sẵn và các ký hiệu người dùng tự dạy thêm. Bản đã dạy đang dùng có 16 ký hiệu, học từ 23 clip. Trang web chạy công khai tại `https://vsl1.lvn123.site`.
2. **Bước ngoặt.** Trong tháng 10/2026, dự án tự huấn luyện mô hình trên video tự thu. Phép đo trên bản ghi chưa học cho 44,8% từ đúng, trong khi mục tiêu đặt trước là 80%. Từ 31/10, dự án đổi hướng: dùng một mô hình nền có sẵn và chỉ học thêm một lớp nhỏ cho từng người dùng.
3. **Cách làm.** Đo trước khi sửa, đặt tiêu chí trước khi đo, ghi lại cả lần thử thất bại, và không gọi kết quả trên dữ liệu đã học là độ chính xác. Mã, phép đo và tài liệu được viết cùng trợ lý lập trình AI, theo yêu cầu và quyết định của chủ dự án.
4. **Giới hạn lớn nhất.** Chưa có phép đo trên một nhóm người ký tách riêng. Ký hiệu đã dạy hiện chỉ đáng tin với chính người đã dạy nó.

## 2. Hệ thống hoạt động như thế nào

### 2.1 Từ camera đến chữ

```text
Camera (trình duyệt hoặc điện thoại)
  -> bộ theo dõi: 76 điểm khung xương ở mỗi khung hình
  -> chuẩn hoá toạ độ
  -> mô hình nền (400 từ đơn) + đầu phân loại của ký hiệu đã dạy
  -> bộ giải mã theo lượt ký: chọn các từ của cả lượt
  -> chữ tiếng Việt trên trang web hoặc ứng dụng
```

1. **Theo dõi tay.** Một mô hình thị giác tìm 76 điểm trên người ký ở mỗi khung hình: 33 điểm thân, 21 điểm mỗi bàn tay, 1 điểm miệng. Máy chủ dùng bộ theo dõi RTMW chạy trên GPU; khi thiếu GPU thì dùng MediaPipe. Phần nhận dạng mặc định chỉ dùng các điểm này, không dùng hình ảnh.
2. **Mô hình nền.** SPOTER (Boháček và Hrúz, 2022) là một Transformer đọc chuỗi điểm khung xương của một ký hiệu và trả về một trong 400 từ. Dự án dùng trọng số đã huấn luyện sẵn từ kho `IvanScopy/VSL400` (6.238.000 tham số, tệp 25 MB) và không huấn luyện lại. Mô hình đọc tối đa 150 khung hình và cần ít nhất 8 khung thấy tay.
3. **Ký hiệu đã dạy.** Mỗi máy có một đầu phân loại riêng: một lớp nhỏ học trên đặc trưng của mô hình nền (mục 2.2).
4. **Giải mã theo lượt ký.** Một lượt ký bắt đầu khi tay vào khung hình hoặc tay đang nghỉ bắt đầu chuyển động, và kết thúc khi tay rời khung 320 ms hoặc đứng yên 1 giây. Trong lượt, cứ 120 ms máy phân loại bốn cửa sổ dài 400, 650, 1.000 và 1.600 ms. Khi lượt kết thúc, máy chọn dãy cửa sổ không chồng nhau có tổng (điểm × độ dài) lớn nhất; đó là câu được viết ra. Nhờ vậy người ký làm được nhiều ký hiệu liền nhau mà không phải hạ tay giữa các ký hiệu.
5. **Từ chối.** Kết quả không đủ chắc thì không được viết. Hệ thống không đoán để lấp chỗ trống.

Mô hình nền là bộ phân loại từ đơn. Câu trên màn hình là dãy các từ đã nhận, chưa phải bản dịch câu VSL tự nhiên.

### 2.2 Dạy thêm một ký hiệu

Người dùng quay một clip ngắn ở trang "Dạy AI", gõ nhãn và xác nhận. Máy chủ làm tiếp các bước sau, không cần khởi động lại:

1. Lưu clip, nhãn và mã băm SHA-256 của tệp.
2. Theo dõi clip bằng ba bộ theo dõi (của máy tính, của điện thoại ở chế độ ảnh và chế độ video) và tạo thêm bản lật gương, để ký hiệu nhận được trên cả hai thiết bị và bằng cả hai tay.
3. Học đầu phân loại: một lớp tuyến tính trên đặc trưng 108 chiều của mô hình nền ghép với quỹ đạo các khớp. Mô hình nền giữ nguyên.
4. Lưu mẫu chuyển động của clip. Khi nhận dạng, một ký hiệu đã dạy chỉ được chấp nhận khi đầu phân loại (điểm từ 0,65) và phép so chuyển động DTW (khoảng cách không quá 0,035) cùng đồng ý.
5. Phát lại clip mới qua đúng bộ giải mã của camera. Clip mà máy không viết lại đúng nhãn của nó thì bị từ chối kèm lý do, để người dùng quay lại.
6. Đổi sang bản mới bằng một thao tác nguyên tử (thay một tệp con trỏ). Bản cũ được giữ để quay lui.
7. Đo độ chắc của từng ký hiệu: với ký hiệu có từ hai clip, máy học một bản thiếu từng clip rồi phát clip đó. Trang Dạy AI hiện ký hiệu nào chắc, yếu hoặc chưa đo được.

Một lượt học sau khi thêm một clip mất 13–16 giây trên máy phát triển (24 clip; đo trước khi có bước 5 và bước 7). Bước 7 chạy sau khi bản mới đã được dùng và mất khoảng 90 giây cho 23 clip.

### 2.3 Ghép bộ ký hiệu nền với ký hiệu đã dạy

Ký hiệu đã dạy luôn được ưu tiên. Bộ ký hiệu nền là phương án dự phòng: nó được hỏi đúng một lần, khi một lượt ký kết thúc mà không có ký hiệu đã dạy nào. Từ của nó chỉ được viết khi xác suất từ 0,915 trở lên và lượt ký dài 1,5 đến 5,5 giây. Hai giới hạn này lấy từ phép đo (mục 4.2).

### 2.4 Đường truyền và thiết bị

- **Trang web.** Trình duyệt gửi hình camera 640×480 đến máy chủ qua WebRTC, rồi nhận về kết quả cùng khung xương để vẽ đè lên hình. Khi WebRTC không nối được, trang tự chuyển sang WebSocket rồi HTTP.
- **Ra internet.** Máy chủ nằm sau mạng gia đình. Trang web đi qua đường hầm Cloudflare; luồng video đi qua máy chủ chuyển tiếp (TURN) của Cloudflare khi hai đầu không nối thẳng được.
- **Android.** Ứng dụng VSL Studio (Kotlin) nhận dạng ngay trên điện thoại bằng cùng lõi Rust, hoặc gửi hình về máy tính. Ứng dụng lấy ký hiệu đã dạy từ máy tính và kiểm SHA-256 trước khi dùng. Giải mã theo lượt ký hiện chỉ có trên máy chủ; ứng dụng vẫn chốt từng từ.

### 2.5 Các phần của mã nguồn

| Phần | Ngôn ngữ | Quy mô ngày 23/11/2026 | Việc |
|---|---|---|---|
| `src/` | Python | 27.963 dòng, 111 tệp | Máy chủ, bộ theo dõi, huấn luyện, giải mã theo lượt ký |
| `src/demo/static/` | JavaScript, HTML, CSS | 13.230 dòng | Trang camera và trang Dạy AI |
| `rust/` | Rust | 3.533 dòng | Lõi nhận dạng dùng chung, cầu nối sang Python và Android |
| `android/` | Kotlin | 6.171 dòng, 24 tệp | Ứng dụng VSL Studio |
| `tests/` | Python, JavaScript | 93 tệp Python (1.277 ca), 4 tệp JavaScript (87 ca) | Kiểm thử tự động |
| `docs/` | Markdown | 25 tài liệu | Mô tả, số đo, giới hạn, báo cáo theo thời điểm |

Máy phát triển là một laptop có GPU RTX 3050 4 GB. Máy chủ của trang công khai cũng chạy trên máy này.

## 3. Quá trình làm

| Thời gian | Việc làm | Kết quả và quyết định |
|---|---|---|
| 12/09 – 10/10/2026 | Khảo sát nguồn dữ liệu VSL và quyền sử dụng. Dựng phòng thí nghiệm chạy trên máy cá nhân: trích 76 điểm bằng MediaPipe, mô hình tự huấn luyện, trang web thử. | Mô hình bảng chữ cái được dùng tạm từ 05/10, gắn nhãn "chưa kiểm định". 572 ca kiểm thử đạt (10/10). |
| 11 – 15/10/2026 | Tự huấn luyện mô hình GRU trên video tự thu: 37 video cá nhân (68 lớp), rồi 426 clip (293 lớp, một người ký). Thử nhận câu ký liền, mô hình câu trên bộ VSL-GH, thêm từ điển QIPEDC (3.474 từ). Đưa mã lên GitHub. | Câu ký liền ngoài đời: 0/19 từ đúng. Mô hình câu VSL-GH với người ký mới: 187/300 câu đã biết, 1/34 câu mới. Thêm QIPEDC làm số lần quay nhận đúng giảm từ 33 xuống 21 trên 77. Không kích hoạt bản có QIPEDC. |
| 16 – 21/10/2026 | Đo lại sạch: mỗi từ giữ ra trọn một bản ghi gốc. Thử DTW (so khớp mẫu), thêm nét mặt, cắt lại clip, mô hình video CNN3D cho tính năng dạy thêm, và hai cách chốt từ mới cho camera. | 13/29 từ đúng (44,8%), mục tiêu đặt trước là 80%: không thay mô hình. Con số 33/77 trước đó được đính chính là thổi phồng, vì nhiều "lần quay khác" là cùng một đoạn phim. CNN3D đúng 11/25, kém GRU (13/25): không kích hoạt. Hai thử nghiệm cho camera ghi "NO-GO". |
| 31/10 – 02/11/2026 | Đổi hướng: mô hình nền SPOTER VSL400 thành mặc định; người dùng dạy thêm ký hiệu qua một đầu phân loại cá nhân, máy tự học và tự kích hoạt. Rà soát toàn bộ mã nguồn (02/11). | Mã chuyển đổi cho kết quả trùng mã gốc (sai lệch lớn nhất bằng 0). Phát hiện bản đã dạy không dùng được trên camera (0/17 cửa sổ), sửa thành 17/17. Rà soát: 406 phát hiện; kiểm lại 84 phát hiện nặng nhất thì không còn mức cao, còn 3 mức trung bình. |
| 05 – 15/11/2026 | Thiết kế rồi viết ứng dụng Android VSL Studio (0.3.0 đến 0.5.0). Chuyển phần nhận dạng sang lõi Rust dùng chung cho máy tính và điện thoại. Kết nối điện thoại qua Cloudflare. Dạy sao cho bền với môi trường và nhận cả hai tay. | Bản chỉ học bằng bộ theo dõi của máy tính nhận 113/120 trên máy tính nhưng 41/120 trên điện thoại mô phỏng. Học thêm bằng bộ theo dõi của điện thoại: 115/120 và 111/120. Ký bằng tay kia: từ 1/8 lên 7/8. Bỏ bản nhận dạng Python trùng với lõi Rust. |
| 18/11/2026 | Cập nhật README và báo cáo. Thử hai cải tiến: chống mất ký hiệu khi camera khựng (P1) và chống từ sai khi tay nghỉ (H1). | P1 sinh từ sai ở 22/54 trường hợp nghỉ tay, vượt ngưỡng 10% đã đặt. H1 không hội tụ. Không phát hành cả hai. |
| 22 – 23/11/2026 | Giải mã theo lượt ký để ký liền nhiều ký hiệu. Tăng tốc lõi và lượt học. Camera qua WebRTC, máy chủ chuyển tiếp, đưa trang ra công khai. Chặn clip yếu và đo độ chắc từng ký hiệu. Bộ ký hiệu nền thành phương án dự phòng. Dọn kho mã. | Số đo ở mục 5. Thay đổi đang chạy trên máy chủ và được đưa lên GitHub ngày 23/11. |

Tính đến 18/11/2026, lịch sử kho mã có 29 commit (từ 14/10): 19 trong tháng 10 và 10 trong tháng 11. Ngày 23/11, thư mục gốc được dọn: 231 script nháp và các tệp rác bị xoá, báo cáo cũ được gom vào `docs/`.

## 4. Cách làm việc

### 4.1 Ai làm gì

- **Chủ dự án** đặt yêu cầu và thứ tự ưu tiên, quay và gắn nhãn clip ký hiệu, quyết định về dữ liệu và quyền dùng, và chọn giữ hay bỏ một hướng. Ví dụ: quyết định "ký hiệu đã dạy luôn thắng bộ ký hiệu nền" là của chủ dự án.
- **Trợ lý lập trình AI** đọc mã, viết mã và kiểm thử, chạy phép đo, viết tài liệu, rồi báo lại số đo cùng giới hạn. 27 trong 29 commit có dòng đồng tác giả ghi rõ việc này (Claude của Anthropic); nhật ký `docs/REPORT.md` cũng ghi tên trợ lý ở các thử nghiệm.
- **Việc chỉ người làm được:** quay clip, thử trước camera thật, thử từ mạng khác, quyết định sao lưu và phát hành. Trợ lý không tự commit, không tự đổi cài đặt hệ thống và không mở các tệp bí mật (khoá, mã ghép nối, `.env`).

### 4.2 Vòng lặp của một thay đổi

1. Tái hiện lỗi và đo nó trên dữ liệu thật, bằng kịch bản phát lại clip.
2. Thử cách sửa ở ngoài sản phẩm, so nhiều phương án trên cùng dữ liệu.
3. Viết vào sản phẩm, kèm kiểm thử tự động.
4. Đo lại bằng chính mã của sản phẩm.
5. Đưa lên máy chủ và phát clip qua đường camera thật.
6. Ghi số đo, ngày đo và điều chưa đo vào tài liệu.

**Ví dụ: lỗi "loạn từ" ngày 23/11/2026.**

- **Triệu chứng.** Khi bật bộ ký hiệu nền, câu có thêm từ lạ chen giữa các ký hiệu đã dạy.
- **Đo.** Phát lại 131 clip từ điển của một người ký khác, 14,4 phút ký ngoài từ vựng và 23 clip đã dạy. Bộ ký hiệu nền khi đó chạy song song với phần đã dạy. Nó viết 72 từ sai trên clip từ điển, 75 từ sai khi ký ngoài từ vựng (5,2 từ mỗi phút) và chen 8 từ vào clip đã dạy.
- **Nguyên nhân.** Bộ ký hiệu nền không có lớp "không phải ký hiệu", nên chuyển động nào cũng được gán một từ.
- **Thử.** Hỏi bộ ký hiệu nền một lần cho cả lượt ký, với nhiều ngưỡng xác suất và độ dài. Ba cách lọc thêm bị bỏ vì số đo không ủng hộ: hỏi hai lần trên lượt ký cắt bớt hai đầu; bỏ lượt ký mà phần đã dạy suýt nhận ra; đòi nhiều cửa sổ cùng nhận một ký hiệu.
- **Sửa.** Viết quy tắc dự phòng vào bộ giải mã, kèm 13 ca kiểm thử mới.
- **Đo lại bằng mã sản phẩm.** 8 từ nền sai (trước là 72); 0 từ khi ký ngoài từ vựng (trước là 75); 0 từ chen vào clip đã dạy (trước là 8). Số clip từ điển được viết đúng tăng từ 44 lên 48.
- **Đường camera thật.** Lần đưa lên đầu tiên còn viết "Dừng lại" lên một cụm đã dạy. Đo tiếp cho thấy các câu trả lời đúng của bộ nền nằm ở xác suất từ 0,915 và dài 1,9–5,2 giây. Giới hạn được siết về 0,915 và 1,5–5,5 giây. Chạy lại, 23 clip đã dạy không còn từ nền nào.
- **Còn lại.** 12 từ nền sai trên 131 clip của người ký khác không lọc được bằng ngưỡng, vì chúng có xác suất ngang từ đúng.

### 4.3 Quy tắc đo

- **Tiêu chí trước, số đo sau.** Trước phép đo quyết định, tiêu chí đạt hay không đạt được viết ra và khoá bằng mã băm. Ví dụ: giao thức của hai thử nghiệm ngày 19 và 20/10 có mã băm SHA-256 ghi trong `docs/REPORT.md`; tệp giao thức gốc đã bị xoá cùng dữ liệu cũ ngày 31/10. Mục tiêu 80% của đợt 16/10 được đặt trước; kết quả 44,8% nên mô hình không được thay.
- **Bộ kiểm tra chỉ đọc một lần.** Bộ đã dùng để quyết định thì bị khoá, không dùng để chỉnh tiếp. Vì vậy `tests/test_isolated_v1.py` không nằm trong lượt chạy thường ngày.
- **Số học thuộc không phải độ chính xác.** "Nhận lại clip đã học" chỉ cho biết đường xử lý chạy đúng. Mỗi con số đều ghi rõ nó đo trên dữ liệu đã học, trên bản ghi giữ ra, hay trên người ký khác.
- **Ghi cả kết quả hỏng và đính chính.** Con số 33/77 được đính chính ngày 16/10. Con số "2,1 từ sai mỗi phút" được sửa thành 5,2 ngày 23/11 vì đã chia nhầm thời lượng.
- **Không nâng ngưỡng để che lỗi.** Ngày 14/10, mô hình khi đó gán từ cho 51/52 video chưa học ở ngưỡng 0,5 và vẫn 22/52 ở ngưỡng 0,99. Cách sửa là dạy thêm mẫu "ký hiệu chưa học"; sau đó 0/39 video chưa học bị gán từ.

### 4.4 Kiểm thử

- **Tự động.** 1.277 ca kiểm thử Python trong 93 tệp, 87 ca kiểm thử trang web chạy bằng Node, 48 hàm kiểm thử trong lõi Rust. Lần chạy đủ ngày 23/11/2026: Python 1.264 đạt, 10 bỏ qua, 3 hỏng vì thiếu tệp dữ liệu cũ đã xoá; trang web 87/87.
- **Đối chiếu khi viết lại.** Phép so DTW trong lõi Rust khớp từng bit với thư viện tham chiếu `tslearn`. Khi tăng tốc lõi từ 53 xuống 15 ms mỗi cửa sổ, kết quả mới giống kết quả cũ từng bit trên 1.454 cửa sổ.
- **Trên đường thật.** Clip được phát vào trang web thay cho camera, hoặc gửi qua WebRTC vào máy chủ đang chạy. Báo cáo luôn ghi rõ đó là clip phát lại, không phải người thật trước camera.

```text
Chạy máy chủ:    .venv\Scripts\python.exe -X utf8 run_app.py
Kiểm thử Python: .venv\Scripts\python.exe -m pytest -q tests --ignore=tests/test_isolated_v1.py
Kiểm thử trang:  node --test tests/frontend_*.test.cjs
```

### 4.5 Phiên bản và dữ liệu

- Mọi mô hình và clip có mã băm SHA-256. Mô hình nền được kiểm băm mỗi lần nạp.
- Bản đã dạy mới chỉ thay bản cũ sau khi tự kiểm tra xong, bằng một thao tác nguyên tử. Trang Dạy AI có nút quay lui.
- Clip chỉ vào huấn luyện khi có nhãn đã xác nhận và quyền sử dụng. Video tải lên để nhận dạng không tự thành dữ liệu huấn luyện.
- Không đưa lên git: video người dùng, tệp mô hình lớn, mã ghép nối, khoá và tệp `.env`. Khi commit chỉ thêm từng đường dẫn cụ thể.
- Kho mã đầy đủ (`VSL`) là riêng tư. Kho công khai `vsl-studio` chỉ có phần giới thiệu và báo cáo, không có mã nguồn, dữ liệu hay trọng số.

### 4.6 Tài liệu đi cùng mã

- Thay đổi nào có số đo thì số đo được ghi vào `docs/`, kèm ngày đo, dữ liệu đo và điều chưa đo.
- `docs/REPORT.md` là nhật ký theo ngày. `docs/BAO_CAO.md` và `docs/BAO_CAO_MA_NGUON_2026-11-02.md` là báo cáo theo thời điểm.
- Trạng thái đang chạy lấy từ `/api/health` và `/api/teach/status`, không lấy từ tài liệu.

## 5. Kết quả đo ngày 22–23/11/2026

| Hạng mục | Trước | Sau | Điều kiện đo |
|---|---|---|---|
| Ký liền 3–5 ký hiệu đã dạy, không nghỉ tay | 120/159 ký hiệu | 145/159 | 40 chuỗi ghép từ clip đã dạy, phát lại |
| Thời gian tính một cửa sổ trong lõi | 53 ms | 15 ms | Kết quả giống từng bit trên 1.454 cửa sổ |
| Một lượt học sau khi thêm một clip | 70–100 giây | 13–16 giây | 24 clip, máy phát triển |
| Khung hình mỗi giây qua địa chỉ công khai | 3,8 (WebSocket) | 24,5 (WebRTC) | Cùng đoạn 5 ký hiệu; 18–20 khi máy đã chạy lâu |
| Từ nền sai trên 131 ký hiệu của người ký khác | 72 | 8 | Phát lại điểm khung xương |
| Từ sai trong 14,4 phút ký ngoài từ vựng | 75 | 0 | Phát lại điểm khung xương |
| Từ nền chen vào 23 clip đã dạy | 8 | 0 | Phát lại; trên đường camera thật cũng là 0 |

Trạng thái của bản đang chạy:

- **Ký hiệu đã dạy.** 16 ký hiệu từ 23 clip. Phát lại clip đã học đúng 21/23. "Ký lại" (học thiếu clip đó rồi phát nó) đúng 5/12. Độ chắc: 1 ký hiệu chắc, 5 yếu, 10 chưa đo được vì chỉ có một clip.
- **Bộ ký hiệu nền với người ký khác.** Qua bộ theo dõi trực tiếp, 131 clip từ điển cho 53 từ nền đúng và 12 từ nền sai. Ở 58 clip máy không viết gì; ở 8 clip phần đã dạy viết nhầm.
- **Đường camera thật.** Năm ký hiệu liền ra đúng "xin chào 1 2 Xin lỗi a". 18 clip từ điển cho 9 đúng, 2 sai, 7 không viết gì.

## 6. Giới hạn và rủi ro còn mở

- Chưa đo trên người ký khác với bộ dữ liệu tách riêng người ký. Trạng thái của hệ thống vẫn là `production_eligible=false`.
- Ký hiệu đã dạy gần như không nhận được ở người khác (2/16, đo ngày 15/11). Trên ký hiệu của người khác, phần đã dạy còn viết nhầm 24 từ trong 131 clip.
- 10 trong 16 ký hiệu đã dạy mới có một clip và 5 ký hiệu yếu. Mỗi ký hiệu cần ít nhất 3 clip, quay ở các buổi khác nhau.
- Bộ ký hiệu nền là bộ phân loại từ đơn. Hệ thống chưa dịch câu VSL tự nhiên; ngữ pháp và nét mặt chưa có trong mô hình.
- Mọi phép đo ngày 22–23/11 dùng clip phát lại. Chưa thử với người thật trước camera thật và chưa thử từ thiết bị ở mạng khác. Ứng dụng Android chưa được dựng lại với các thay đổi mới.
- Máy phát triển đã gặp lỗi phần cứng nhiều lần. Mã đã nằm trên GitHub, nhưng các clip đã dạy (`data/teach`) chỉ nằm trên máy này và cần sao lưu riêng.
- Máy chủ không tự chạy lại sau khi máy khởi động lại.

## 7. Bài học

1. **Dữ liệu quyết định trước mô hình.** Với một người ký và 203/258 từ chỉ có một bản ghi gốc, không mô hình tự huấn luyện nào đạt mục tiêu. Dùng mô hình nền có sẵn giải quyết được phần 400 từ.
2. **Phải đo đúng đường sẽ dùng.** Phát lại nguyên clip thì máy nhận đúng, nhưng trên camera 0/17 cửa sổ dùng được (01/11). Từ đó mọi phép đo phát lại qua đúng bộ giải mã của camera.
3. **Ngưỡng không thay được lớp "không biết".** Hai lần, ở hai mô hình khác nhau, nâng ngưỡng không loại được từ sai (22/52 ở ngưỡng 0,99; 12 từ nền sai đạt xác suất tới 0,929).
4. **Học và dùng phải cùng bộ theo dõi.** Bản học bằng bộ theo dõi của máy tính tụt từ 113/120 xuống 41/120 trên điện thoại.
5. **Viết lại cho nhanh thì phải chứng minh kết quả không đổi.** Đối chiếu từng bit cho phép tăng tốc 3,5 lần mà không cần đo lại độ đúng.
6. **Ghi lại thử nghiệm thất bại.** Các mục "dừng" và "NO-GO" giúp không thử lại cùng một hướng.

## 8. Việc tiếp theo

1. Quay lại clip cho 5 ký hiệu yếu và 10 ký hiệu mới có một clip.
2. Thử với người thật trước camera và từ mạng di động.
3. Sao lưu các clip đã dạy ra ngoài máy và cho máy chủ tự khởi động.
4. Dựng lại ứng dụng Android với lõi mới.
5. Đo trên người ký khác bằng dữ liệu tách riêng người ký.
6. Hướng dài hơn, đã có khung mã (`src/train/direct_sequence.py`): học thẳng từ chuỗi điểm khung xương sang câu tiếng Việt đã kiểm chứng.

Mốc công bố dự kiến: thứ Năm 26/11/2026.

## 9. Nguồn số liệu

| Nội dung | Nguồn |
|---|---|
| Lịch sử commit, dòng đồng tác giả | `git log` nhánh `main` (29 commit, 14/10–18/11/2026) |
| Giai đoạn tự huấn luyện và các thử nghiệm dừng | `docs/BAO_CAO.md`, `docs/REPORT.md` |
| Mô hình nền, giải mã theo lượt ký, bộ ký hiệu nền, tốc độ lõi | `docs/FOUNDATION.md` |
| Dạy thêm, chặn clip yếu, độ chắc, bền với môi trường | `docs/AUTO_TEACH.md` |
| WebRTC và máy chủ chuyển tiếp | `docs/WEBRTC_DEEPMIND_ROUTE.md` |
| Rà soát mã nguồn | `docs/BAO_CAO_MA_NGUON_2026-11-02.md` |
| Nguồn dữ liệu và quyền dùng | `docs/DATA_SOURCES.md` |
| Thử nghiệm P1, H1 | `README.md` và báo cáo ngày 18/11/2026 |
| Quy mô mã và kiểm thử | Đếm trên thư mục làm việc và lần chạy kiểm thử ngày 23/11/2026 |

## 10. Thuật ngữ

- **Ký hiệu:** một động tác tay mang nghĩa một từ trong ngôn ngữ ký hiệu.
- **Người ký:** người làm ký hiệu trước camera.
- **Bộ theo dõi (tracker):** mô hình tìm vị trí các điểm trên thân và bàn tay trong từng khung hình.
- **Điểm khung xương (landmark):** một điểm do bộ theo dõi trả về, ví dụ đầu ngón trỏ.
- **Mô hình nền:** mô hình đã huấn luyện sẵn mà dự án dùng lại, ở đây nhận 400 từ đơn.
- **Đầu phân loại:** lớp nhỏ học thêm trên mô hình nền để nhận các ký hiệu người dùng tự dạy.
- **DTW:** phép so hai chuyển động có tốc độ khác nhau bằng cách căn chỉnh chúng theo thời gian.
- **Cửa sổ:** một đoạn ngắn các khung hình liên tiếp được đưa cho mô hình.
- **Lượt ký:** quãng từ lúc tay bắt đầu ký đến lúc tay hạ xuống hoặc đứng yên.
- **WebRTC, TURN:** cách truyền video trực tiếp giữa trình duyệt và máy chủ; TURN là máy chủ chuyển tiếp khi hai đầu không nối thẳng được.
- **SHA-256:** mã băm của một tệp; tệp đổi một bit thì mã đổi.
- **Tách riêng người ký:** người ký trong dữ liệu kiểm tra không có mặt trong dữ liệu học.
