# Test case — Công cụ và hồ sơ chuyển đổi

Tài liệu này chuyển các yêu cầu trong [đặc tả chuyển đổi](conversion.md) thành test case chức năng và cấu hình. Đây là kế hoạch kiểm thử, chưa phải kết quả chạy thực tế.

## 1. Quy ước

- **Trạng thái ban đầu:** Chưa chạy. Tester cập nhật thành PASS/FAIL/BLOCKED và ghi bằng chứng sau khi thực thi.
- **Dữ liệu:** Dùng ID duy nhất cho thư viện, tài liệu, công cụ và hồ sơ trong từng lượt chạy. Không phụ thuộc dữ liệu tồn tại sẵn.
- **Cô lập:** Dọn dữ liệu và sản phẩm thử nghiệm sau mỗi nhóm; không xóa dữ liệu dùng chung.
- **Kiểm chứng:** Xác nhận qua UI, API, DB, hàng đợi/tác vụ, filesystem và log tương ứng. Không chỉ dựa vào thông báo trên UI.
- **Giới hạn:** Các giá trị cài đặt ghi trong bảng dưới đây là cấu hình test đề xuất; nếu môi trường dùng giá trị khác, ghi lại giá trị thực tế trước khi chạy.

## 2. Cấu hình và dữ liệu chuẩn

| Thành phần | Cấu hình chuẩn cho test |
| --- | --- |
| Toàn hệ thống | `build.converter.concurrency=4` |
| Công cụ CLI A | `maxConcurrentJobs=2`, `timeoutSeconds=300`, hỗ trợ PDF→PDF, PDF→SVG và SVG; `thumbnailSupported=true` |
| Công cụ CLI B | `maxConcurrentJobs=1`, `timeoutSeconds=300`, hỗ trợ tập định dạng độc lập để kiểm tra phân bổ |
| Hồ sơ PDF | Dùng công cụ A, `outputFormat=PDF`, tham số mặc định riêng của hồ sơ, `timeoutSeconds=null` |
| Hồ sơ SVG | Dùng công cụ A, `outputFormat=SVG`, SVG 1.1, nhúng ảnh, JPEG quality 80, `timeoutSeconds=null` |
| Thư viện chuẩn | Hồ sơ PDF; `conversionTiming=PRECONVERT`; `updateCheckMode=METADATA`; `generateThumbnail=false`; `normalConversionTarget=1` |
| Tài liệu thử | PDF hợp lệ nhiều trang, PDF mã hóa/cấm trích xuất, PDF cùng nội dung nhưng metadata khác, PDF giữ metadata nhưng đổi nội dung, một định dạng không hỗ trợ |
| Bộ chuyển đổi thử | Stub/fixture có thể tạo đầu ra hợp lệ, trì hoãn theo thời gian cấu hình, trả mã lỗi theo kịch bản, ghi argv/đầu ra và tạo thumbnail xác định |

> Không dùng fixture thay thế để kết luận về khả năng tương thích thực tế của OSDC. Các test tích hợp PDF/SVG, bảo mật PDF và thumbnail cần chạy thêm với phiên bản OSDC được hỗ trợ.

## 3. Test case

### A. Cấu hình công cụ, hồ sơ và tham số hiệu lực

#### TC-CFG-001 — Giá trị mặc định và hiển thị cài đặt công cụ

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có quyền quản trị; DB đã áp dụng schema.
- **Dữ liệu/cấu hình:** Tạo công cụ mới, bỏ trống `maxConcurrentJobs` và `timeoutSeconds`.
- **Bước thực hiện:**
  1. Gọi `GET /api/converter-engines/defaults`.
  2. Mở giao diện tạo công cụ mới.
  3. Tạo công cụ không gửi hai trường giới hạn đồng thời và timeout.
  4. Mở danh sách, chi tiết và JSON export của công cụ.
- **Kết quả mong đợi:**
  - `convertUrl` trong defaults là chuỗi rỗng; UI đường dẫn thực thi trống.
  - Placeholder lệnh mẫu chỉ để minh họa, không được lưu làm giá trị.
  - Công cụ mới có `maxConcurrentJobs=1`, `timeoutSeconds=300`.
  - Hai giá trị được trả về nhất quán trong UI, API và export.
- **Trạng thái:** Chưa chạy

#### TC-CFG-002 — Placeholder không được lưu và đường dẫn thực thi vẫn bắt buộc

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Đang tạo công cụ mới.
- **Dữ liệu/cấu hình:** Placeholder `C:\Tools\SampleConverter.exe -in {input} -out {output}`.
- **Bước thực hiện:**
  1. Không nhập đường dẫn thực thi và gửi form.
  2. Nhập đường dẫn thực tế cùng đối số hợp lệ rồi lưu.
  3. Mở chỉnh sửa công cụ và tải lại trang.
- **Kết quả mong đợi:**
  - Không cho lưu nếu thiếu đường dẫn thực thi.
  - Placeholder không được gửi/lưu ngầm.
  - Khi chỉnh sửa, hiển thị đúng giá trị đã lưu.
  - Giá trị mặc định OS-specific khi khởi tạo công cụ chuẩn và lệnh đã đăng ký trước đó không bị thay đổi.
- **Trạng thái:** Chưa chạy

#### TC-CFG-003 — Giới hạn đồng thời của công cụ: biên hợp lệ và không hợp lệ

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có quyền quản trị.
- **Dữ liệu/cấu hình:**
  - Hợp lệ: `1`, `64`.
  - Không hợp lệ: `0`, `65`, `-1`, `1.5`, `"2"`, chuỗi rỗng không được phép, giá trị không phải số nguyên.
- **Bước thực hiện:**
  1. Tạo/cập nhật công cụ với từng giá trị hợp lệ.
  2. Gửi lần lượt từng giá trị không hợp lệ qua API.
  3. Cập nhật công cụ với payload bỏ trường và với `null`.
- **Kết quả mong đợi:**
  - Chấp nhận số nguyên trong khoảng 1–64.
  - Từ chối giá trị sai bằng lỗi xác thực rõ ràng; DB không bị đổi.
  - Bỏ trường/null khi cập nhật giữ nguyên giá trị hiện tại.
  - Bỏ trường khi tạo dùng mặc định 1.
- **Trạng thái:** Chưa chạy

#### TC-CFG-004 — Timeout của công cụ: biên hợp lệ, mặc định và ngữ nghĩa cập nhật

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có quyền quản trị.
- **Dữ liệu/cấu hình:** Hợp lệ `1`, `2147483647`; không hợp lệ `0`, số âm, `2147483648`, số thập phân, chuỗi.
- **Bước thực hiện:**
  1. Tạo công cụ với timeout bị bỏ qua và với null.
  2. Tạo/cập nhật bằng hai giá trị biên hợp lệ.
  3. Thử lần lượt các giá trị không hợp lệ.
  4. Đặt timeout hiện tại thành 120, cập nhật payload bỏ trường rồi payload null.
- **Kết quả mong đợi:**
  - Tạo với bỏ trường/null cho kết quả 300 giây.
  - Chấp nhận số nguyên 1–2147483647; từ chối các giá trị còn lại và giữ nguyên DB khi lỗi.
  - Cập nhật bỏ trường/null giữ nguyên 120 giây.
- **Trạng thái:** Chưa chạy

#### TC-CFG-005 — Hiển thị và cập nhật các trường cấu hình công cụ

- **Ưu tiên:** Trung bình
- **Tiền điều kiện:** Có ít nhất một công cụ đã lưu.
- **Bước thực hiện:**
  1. Kiểm tra màn hình chỉnh sửa, danh sách và API chi tiết.
  2. Export JSON, import lại trong môi trường test cô lập.
  3. Thay đổi `maxConcurrentJobs` và `timeoutSeconds`, sau đó đọc lại.
- **Kết quả mong đợi:**
  - Các trường xuất hiện nhất quán ở UI/list/detail/export.
  - Giá trị được giữ nguyên qua export/import và cập nhật.
  - Cột DB công cụ hiện có nhận giá trị đồng thời mặc định 1; timeout hiện có là 300 giây.
- **Trạng thái:** Chưa chạy

#### TC-CFG-006 — Hồ sơ xác định duy nhất định dạng đầu ra

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Công cụ A hỗ trợ PDF và SVG; có quyền quản trị.
- **Bước thực hiện:**
  1. Tạo hồ sơ PDF và hồ sơ SVG trên cùng công cụ.
  2. Thử lưu hồ sơ với định dạng đầu ra không được công cụ hỗ trợ.
  3. Thử cấu hình thư viện chọn riêng định dạng đầu ra ngoài hồ sơ.
  4. Chọn trình xem không tương thích với định dạng hồ sơ và lưu/build.
- **Kết quả mong đợi:**
  - Mỗi hồ sơ có đúng một `outputFormat`.
  - Không chấp nhận định dạng ngoài khả năng của công cụ.
  - Không có lựa chọn độc lập để thư viện đổi định dạng đầu ra.
  - Tính tương thích giữa định dạng hồ sơ và trình xem được kiểm tra khi lưu và khi build; lỗi có thông báo rõ.
- **Trạng thái:** Chưa chạy

#### TC-CFG-007 — Thứ tự hợp nhất tham số và không trộn tham số của engine

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có hồ sơ với tham số mặc định P1/P2; engine có một tham số riêng không nằm trong hồ sơ.
- **Dữ liệu/cấu hình:** Thư viện ghi đè P1 và thêm P3 trong `parameterOverrides`.
- **Bước thực hiện:**
  1. Mở “Cấu hình tham số” và xem giá trị hiệu lực.
  2. Build một tài liệu và lấy cấu hình/lệnh thực tế từ log.
  3. Đổi P1, rồi build lại bằng DIFF.
- **Kết quả mong đợi:**
  - Giá trị hiệu lực gồm P1 đã ghi đè, P2 mặc định và P3 bổ sung.
  - Danh sách tham số riêng của engine không bị tự động trộn vào.
  - Thay đổi tham số hiệu lực làm tài liệu cần chuyển đổi lại; cấu hình build đã bắt đầu không bị thay đổi giữa chừng.
- **Trạng thái:** Chưa chạy

#### TC-CFG-008 — Xác thực tham số bắt buộc, chỉ đọc, boolean và xung đột

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Hồ sơ có tham số bắt buộc, chỉ đọc và boolean.
- **Bước thực hiện:**
  1. Xóa tham số bắt buộc hoặc gửi giá trị sai kiểu boolean.
  2. Thử sửa tham số chỉ đọc.
  3. Gửi chuỗi rỗng và hai tham số xung đột cùng tạo một đối số.
  4. Quan sát form và gửi qua API.
- **Kết quả mong đợi:**
  - Từ chối cấu hình thiếu/sai kiểu hoặc có đối số xung đột; không chạy lệnh không hợp lệ.
  - Giá trị chỉ đọc không thể bị sửa từ UI hoặc API.
  - Form đặt tiêu đề “Cấu hình tham số”; trường sửa được nền trắng, chỉ đọc nền xám; mục điều chỉnh có viền xanh và thao tác đặt lại.
- **Trạng thái:** Chưa chạy

#### TC-CFG-009 — Chữ ký chuyển đổi và các thay đổi không cần chuyển đổi lại

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có tài liệu đã chuyển đổi thành công và bật theo dõi chữ ký/diff.
- **Bước thực hiện:**
  1. Lưu lại cùng cấu hình; build DIFF.
  2. Đổi trình xem nhưng giữ định dạng đầu ra; build DIFF.
  3. Đổi `outputFormat` hoặc giá trị tham số hiệu lực; build DIFF.
  4. Đổi timeout hoặc số worker đồng thời; build DIFF.
- **Kết quả mong đợi:**
  - Hai trường hợp 1–2 không gây chuyển đổi lại không cần thiết.
  - Đổi định dạng hoặc tham số hiệu lực thì chuyển đổi lại.
  - Chỉ đổi timeout hoặc số lượng đồng thời không làm đổi chữ ký/chuyển đổi lại.
  - Cấu hình đầu ra được cố định theo snapshot lúc bắt đầu build.
- **Trạng thái:** Chưa chạy

### B. PDF, định dạng đầu ra và xử lý cùng định dạng

#### TC-PDF-001 — Khởi tạo công cụ OSDC chuẩn có hỗ trợ PDF

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có OSDC chuẩn mới hoặc hiện có.
- **Bước thực hiện:**
  1. Đăng ký mới công cụ OSDC chuẩn.
  2. Khởi động ứng dụng với dữ liệu công cụ chuẩn hiện có.
  3. Kiểm tra danh sách định dạng đầu vào.
- **Kết quả mong đợi:** PDF có trong định dạng đầu vào của OSDC mới và được bổ sung khi cập nhật công cụ chuẩn hiện có; không làm thay đổi các lệnh đăng ký trước.
- **Trạng thái:** Chưa chạy

#### TC-PDF-002 — Chuyển PDF gốc thành SVG nhiều trang

- **Ưu tiên:** Cao
- **Tiền điều kiện:** OSDC hỗ trợ PDF→SVG; có PDF hợp lệ nhiều trang; hồ sơ SVG và trình xem SVG đã cấu hình.
- **Bước thực hiện:**
  1. Build thư viện với đầu ra SVG.
  2. Kiểm tra số lượng và nội dung SVG theo từng trang.
  3. Kiểm tra thông tin tài liệu SVG và trình xem được gọi.
- **Kết quả mong đợi:** PDF được chuyển thành SVG theo từng trang; thông tin tài liệu được tạo và chuyển cho đúng trình xem.
- **Trạng thái:** Chưa chạy

#### TC-PDF-003 — Chuyển định dạng PDF sang SVG không cần bản gốc thay đổi

- **Ưu tiên:** Cao
- **Tiền điều kiện:** PDF đã được chuyển thành PDF thành công; metadata và nội dung bản gốc không thay đổi.
- **Bước thực hiện:**
  1. Đổi hồ sơ/đầu ra sang SVG.
  2. Chạy DIFF.
  3. Kiểm tra trạng thái và đầu ra tài liệu.
- **Kết quả mong đợi:** Tài liệu được chuyển đổi lại dù bản gốc không đổi.
- **Trạng thái:** Chưa chạy

#### TC-PDF-004 — Khôi phục tài liệu trước đây bị bỏ qua do định dạng

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Một tài liệu trước đây bị skip vì định dạng chưa hỗ trợ; bản gốc vẫn tồn tại và hiện engine hỗ trợ.
- **Bước thực hiện:**
  1. Bật cấu hình đầu vào/đầu ra hiện có thể xử lý tài liệu.
  2. Chạy DIFF.
  3. Lặp lại với các trường hợp tài liệu bị xóa, bị loại khỏi phát hành và vẫn không hỗ trợ.
- **Kết quả mong đợi:**
  - Tài liệu trước đây bị skip được thử lại trong DIFF nếu hiện đã xử lý được.
  - Tài liệu đã xóa/bị loại khỏi phát hành hoặc vẫn không hỗ trợ không thuộc diện phục hồi.
  - PRECONVERT, BACKGROUND và ON_DEMAND dùng cùng quy tắc xác định hỗ trợ định dạng; chuyển đổi sau phát hành áp dụng khi tái phát hành cấu hình mới.
- **Trạng thái:** Chưa chạy

#### TC-PDF-005 — PDF bị mã hóa hoặc hạn chế quyền

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có PDF mã hóa hoặc cấm sao chép/trích xuất; dùng OSDC được hỗ trợ.
- **Bước thực hiện:** Chuyển đổi tài liệu và kiểm tra task state, lý do lỗi và log.
- **Kết quả mong đợi:** Ghi nhận như lỗi chuyển đổi thông thường kèm lý do từ engine; không báo thành công giả hoặc làm hỏng toàn bộ build theo cách không được đặc tả.
- **Trạng thái:** Chưa chạy

#### TC-PDF-006 — Bỏ qua chuyển đổi khi bản gốc cùng định dạng đầu ra

- **Ưu tiên:** Cao
- **Tiền điều kiện:** `skipSameFormat=true`; PDF gốc và outputFormat đều là PDF.
- **Bước thực hiện:**
  1. Build thư viện.
  2. Kiểm tra trạng thái conversion task và tệp PDF được sao chép.
  3. Kiểm tra đăng ký tìm kiếm và dữ liệu đầu vào AI.
  4. Gây lỗi ở bước sao chép hoặc đăng ký chỉ mục trong môi trường kiểm thử.
- **Kết quả mong đợi:**
  - Task ở trạng thái `SKIPPED`, không chuyển trạng thái thành công thông thường.
  - Dùng bản sao PDF gốc; tìm kiếm và AI input vẫn được duy trì.
  - Lỗi sao chép/đăng ký chỉ mục đi theo luồng xử lý lỗi.
  - Khi PDF→SVG, tùy chọn bỏ qua cùng định dạng không áp dụng.
- **Trạng thái:** Chưa chạy

### C. Hồ sơ SVG và cập nhật dữ liệu

#### TC-SVG-001 — Hồ sơ PDF và SVG tiêu chuẩn cùng tồn tại

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Khởi chạy seed/initializer dữ liệu chuẩn.
- **Bước thực hiện:**
  1. Khởi động lần đầu và kiểm tra hồ sơ chuẩn.
  2. Xác nhận PDF vẫn là lựa chọn mặc định.
  3. Vô hiệu hóa hồ sơ SVG hoặc tạo hồ sơ trùng tên, khởi động lại nhiều lần.
- **Kết quả mong đợi:** PDF và SVG cùng tồn tại; SVG chỉ được thêm cho engine CLI hỗ trợ SVG; giữ nguyên hồ sơ trùng tên/vô hiệu hóa; không tạo trùng hoặc ghi đè mỗi lần khởi động.
- **Trạng thái:** Chưa chạy

#### TC-SVG-002 — Tham số SVG tiêu chuẩn và thuộc tính ảnh

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Hồ sơ SVG tiêu chuẩn được tạo; công cụ hỗ trợ SVG.
- **Bước thực hiện:** Kiểm tra các tham số và tạo tài liệu có ảnh raster; xem nội dung SVG, manifest và output offline.
- **Kết quả mong đợi:**
  - Giá trị: `svgver=SVG1.1`, `svgip=0` (chỉ đọc), `svgic=0`, `svgjq=80`, `docpagenumbercorrection=false`, `xlscontinuepagenumber=false`, `pptnotes=false`, `extlevel=2`.
  - SVG được tạo theo trang; manifest có tổng trang và kích thước.
  - Tất cả ảnh được nhúng để sử dụng offline; phân trang và tên do nền tảng chuyển đổi quản lý.
- **Trạng thái:** Chưa chạy

#### TC-DB-001 — Migration schema cho môi trường `validate`

- **Ưu tiên:** Cao
- **Tiền điều kiện:** DB H2 hiện hữu; ứng dụng dùng `ddl-auto=validate`; có bản sao DB để phục hồi.
- **Bước thực hiện:**
  1. Chạy ứng dụng trước khi migration và xác nhận schema cũ thiếu cột được phát hiện.
  2. Dừng ứng dụng, sao lưu DB.
  3. Chạy các lệnh ALTER TABLE trong đặc tả cho `max_concurrent_jobs` và `timeout_seconds`.
  4. Chạy migration lần nữa, sau đó khởi động phiên bản mới.
- **Kết quả mong đợi:**
  - Có thể chạy lại migration mà không lỗi và không mất hàng dữ liệu.
  - Engine hiện có nhận `max_concurrent_jobs=1`, `timeout_seconds=300`; hồ sơ hiện có nhận timeout null (kế thừa).
  - `validate` khởi động thành công sau migration.
  - `ddl-auto=update` tự thêm các cột tương ứng.
- **Trạng thái:** Chưa chạy

### D. Giới hạn đồng thời, hàng đợi và phân bổ công bằng

#### TC-CONC-001 — Giới hạn toàn hệ thống và giới hạn từng engine

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có ít nhất 2 thư viện và engine A/B; bộ chuyển đổi test giữ tiến trình ở trạng thái chạy để đo đồng thời.
- **Dữ liệu/cấu hình:** Global=4; A=2; B=1; tạo nhiều tác vụ trên các thư viện cùng engine ID và khác engine ID.
- **Bước thực hiện:**
  1. Đưa số tác vụ vượt giới hạn vào hàng đợi.
  2. Ghi nhận số tiến trình đang chạy và số slot đã đặt.
  3. Hoàn tất từng tác vụ và quan sát hàng đợi.
- **Kết quả mong đợi:**
  - Không vượt giới hạn toàn hệ thống 4 hoặc A=2/B=1.
  - Các thư viện dùng cùng engine ID chia sẻ slot; engine ID khác nhau có slot riêng nhưng vẫn chịu global cap.
  - Slot đã đặt nhưng chưa khởi chạy vẫn được tính vào giới hạn.
  - Slot được giải phóng sau thành công, lỗi, timeout, hủy hoàn tất hoặc lỗi đưa vào hàng đợi.
- **Trạng thái:** Chưa chạy

#### TC-CONC-002 — Giá trị toàn hệ thống động và khởi tạo từ cấu hình

- **Ưu tiên:** Cao
- **Tiền điều kiện:** DB cài đặt hiện có thể đọc; có tác vụ chờ và tác vụ đang chạy.
- **Bước thực hiện:**
  1. Với DB chưa có cấu hình, khởi động bằng giá trị cấu hình khởi động và kiểm tra giá trị được ghi nhận lần đầu.
  2. Thay đổi giới hạn DB từ 4 xuống 2 rồi lên 5 mà không restart.
  3. Chờ các nhịp scheduler bình thường.
  4. Kiểm thử giá trị DB bị xóa, không hợp lệ và lỗi truy vấn.
- **Kết quả mong đợi:**
  - Chỉ lần đầu lấy giá trị khởi động; sau đó DB hợp lệ được ưu tiên.
  - Giá trị hợp lệ từ 1 đến 64 được áp dụng ở lần kiểm tra tiếp theo (thường khoảng 1 giây), không restart.
  - Giảm giới hạn không hủy việc đang chạy; không khởi chạy mới cho đến khi số chạy thấp hơn giới hạn mới.
  - DB setting mất/không hợp lệ quay về cấu hình khởi động; lỗi đọc DB khiến tác vụ mới chờ, không chạy vượt giới hạn.
- **Trạng thái:** Chưa chạy

#### TC-CONC-003 — Trạng thái QUEUED không đổi thời điểm và số lần thử khi chờ slot

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Đã đạt giới hạn engine hoặc global; có tác vụ đang chờ.
- **Bước thực hiện:**
  1. Ghi lại trạng thái, thời điểm bắt đầu và số lần thử của tác vụ.
  2. Để tác vụ chờ qua nhiều chu kỳ scheduler.
  3. Giải phóng một slot.
- **Kết quả mong đợi:**
  - Khi chờ slot tác vụ giữ trạng thái `QUEUED`; thời điểm bắt đầu và số lần thử không thay đổi.
  - Khi có slot, tác vụ khả thi được bắt đầu theo chính sách ưu tiên.
- **Trạng thái:** Chưa chạy

#### TC-CONC-004 — Bỏ qua engine đã đầy và tiếp tục xử lý tác vụ khác

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Engine A hết slot; có tác vụ của A đứng trước trong hàng đợi và tác vụ dùng B có thể chạy.
- **Bước thực hiện:** Đưa cả hai tác vụ vào scheduler và chờ lượt xử lý tiếp theo.
- **Kết quả mong đợi:** Scheduler không bị chặn bởi tác vụ đang chờ A; tác vụ dùng B hoặc thư viện khác khả thi vẫn được chạy.
- **Trạng thái:** Chưa chạy

#### TC-CONC-005 — Ưu tiên, mục tiêu thường và công bằng giữa thư viện

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Global=4; thư viện A/B/C mỗi thư viện có hàng đợi thông thường; `normalConversionTarget=2`; tác vụ có thời lượng đủ để quan sát.
- **Bước thực hiện:**
  1. Chỉ có A chờ và xác nhận A sử dụng các slot trống.
  2. Trong lúc A đang chạy, thêm B/C với mục tiêu 2.
  3. Ghi nhận lượt cấp phát qua nhiều lần hoàn tất.
  4. Thêm yêu cầu xem ON_DEMAND và tác vụ nền đã tăng ưu tiên do yêu cầu xem.
- **Kết quả mong đợi:**
  - Khi không cạnh tranh, một thư viện có thể dùng toàn bộ slot trống; không có slot build dành riêng.
  - Thư viện chưa đạt mục tiêu được ưu tiên theo tỷ lệ running/target thấp nhất; hòa thì thư viện được phân bổ lâu nhất trước; bên trong thư viện theo thứ tự tiếp nhận.
  - Khi mọi thư viện đạt mục tiêu, slot dư chia theo thứ tự tiếp nhận giữa PRECONVERT và BACKGROUND.
  - Phân bổ không vượt global cap; tiếp tục công bằng theo thời gian, ví dụ 2/1/1 cho A/B/C.
  - Yêu cầu xem được ưu tiên hơn mục tiêu thường; sau khi xử lý xong, bổ sung lại phần thiếu.
- **Trạng thái:** Chưa chạy

#### TC-CONC-006 — Cập nhật `normalConversionTarget` không gây build lại

- **Ưu tiên:** Trung bình
- **Tiền điều kiện:** Thư viện dùng PRECONVERT hoặc BACKGROUND, có hàng đợi tác vụ.
- **Dữ liệu/cấu hình:** `normalConversionTarget` nhận 1–64, mặc định 1.
- **Bước thực hiện:**
  1. Thử lưu 1 và 64; xác nhận từ chối 0, 65, số thập phân.
  2. Đổi target khi có tác vụ chờ.
  3. Chuyển sang ON_DEMAND rồi trở lại BACKGROUND.
  4. Quan sát form, DB, lượt phân bổ và chỉ mục/AI.
- **Kết quả mong đợi:**
  - Tham số được xác thực đúng phạm vi.
  - Thay đổi có hiệu lực từ lần phân bổ sau khi lưu, không cần build/restart/reconvert.
  - Ẩn khi ON_DEMAND, nhưng giữ nguyên giá trị form/DB; hiển thị lại khi trở về PRECONVERT/BACKGROUND.
  - Không làm đổi thế hệ chỉ mục nội dung/AI hoặc mất tác vụ/kết quả đã có.
- **Trạng thái:** Chưa chạy

#### TC-CONC-007 — Hủy tác vụ đang chạy và khôi phục khi restart

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có task đang chạy và task khác chờ; ghi nhận số slot trước thao tác.
- **Bước thực hiện:**
  1. Gửi yêu cầu hủy tác vụ đang chạy.
  2. Quan sát slot trước khi process kết thúc và sau khi process kết thúc.
  3. Restart ứng dụng khi có task chưa hoàn tất.
- **Kết quả mong đợi:**
  - Không giải phóng slot chỉ vì Future nhận tín hiệu hủy; chỉ giải phóng sau khi xử lý thực sự kết thúc.
  - Khi restart, xử lý job chưa hoàn tất theo quy tắc thất bại hiện có; không để slot bị rò rỉ.
- **Trạng thái:** Chưa chạy

#### TC-CONC-008 — Không hỗ trợ nhiều instance cùng chia sẻ DB

- **Ưu tiên:** Thấp
- **Tiền điều kiện:** Môi trường kiểm thử có thể chạy hai instance dùng chung DB.
- **Bước thực hiện:** Chạy đồng thời hai instance và kiểm tra tài liệu/giới hạn vận hành.
- **Kết quả mong đợi:** Hệ thống không tuyên bố hỗ trợ đồng thời nhiều instance chia sẻ DB; giới hạn này được nêu rõ trong tài liệu vận hành. Không dùng kết quả chạy song song để khẳng định global cap xuyên instance.
- **Trạng thái:** Chưa chạy

### E. Timeout chuyển đổi

#### TC-TIME-001 — Thứ tự chọn timeout và kế thừa của hồ sơ

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Engine A có `timeoutSeconds=90`; có hồ sơ được quản trị.
- **Bước thực hiện:**
  1. Tạo hồ sơ `timeoutSeconds=null`.
  2. Tạo hồ sơ `timeoutSeconds=30`.
  3. Gửi tác vụ chuyển đổi chậm hơn 30/90 giây theo từng hồ sơ.
- **Kết quả mong đợi:**
  - Hồ sơ null kế thừa 90 giây từ engine.
  - Hồ sơ chỉ định 30 giây dùng timeout 30 giây.
  - Thứ tự áp dụng là hồ sơ → engine → mặc định 300 giây.
  - Timeout được thiết lập cho mỗi tệp ngay trước khi bắt đầu chuyển đổi ngoài.
- **Trạng thái:** Chưa chạy

#### TC-TIME-002 — Giá trị timeout hồ sơ không hợp lệ và khôi phục kế thừa

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Engine có timeout hợp lệ.
- **Dữ liệu/cấu hình:** Hồ sơ hợp lệ 1, 2147483647, null; không hợp lệ 0, -1, 2147483648, 1.5, chuỗi.
- **Bước thực hiện:**
  1. Tạo/cập nhật hồ sơ với từng giá trị.
  2. Trong UI để trống timeout, lưu và xem lại.
- **Kết quả mong đợi:**
  - Chấp nhận số nguyên 1–2147483647 hoặc null; từ chối các giá trị khác.
  - Khi null/để trống, UI ghi “Kế thừa cài đặt công cụ chuyển đổi” và hiển thị giá trị hiện tại của engine.
  - Tạo/cập nhật với null hoặc bỏ trường trả hồ sơ về kế thừa engine.
- **Trạng thái:** Chưa chạy

#### TC-TIME-003 — Áp dụng timeout mới chỉ cho lượt bắt đầu sau khi lưu

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có tác vụ chậm đang chạy và một tác vụ khác chưa bắt đầu; cấu hình timeout có thể cập nhật.
- **Bước thực hiện:**
  1. Bắt đầu lượt thứ nhất với timeout 120 giây.
  2. Đổi timeout thành 5 giây trong DB/API trong khi lượt thứ nhất đang chạy.
  3. Khởi chạy lượt thứ hai và thử lại một lượt lỗi.
- **Kết quả mong đợi:**
  - Lượt đang chạy giữ nguyên thời hạn 120 giây.
  - Lượt mới và retry bắt đầu sau khi lưu dùng 5 giây mà không restart.
  - Snapshot đầu ra của build không đổi; chỉ timeout được đọc lại từ cấu hình mới nhất.
- **Trạng thái:** Chưa chạy

#### TC-TIME-004 — Thay engine trong hồ sơ giữa build và lỗi lấy cấu hình

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Build đã snapshot engine A; hồ sơ sau đó đổi sang engine B. Có thể mô phỏng DB lỗi/giá trị cấu hình không hợp lệ.
- **Bước thực hiện:**
  1. Bắt đầu build bằng engine A.
  2. Đổi engine trong hồ sơ trước khi lượt kế tiếp bắt đầu.
  3. Gây lỗi đọc DB hoặc timeout invalid ngay trước khi bắt đầu lượt khác.
- **Kết quả mong đợi:**
  - Chỉ dùng timeout của hồ sơ khớp với engine được chọn trong snapshot build.
  - Không chạy không giới hạn nếu DB lỗi/cấu hình sai; chuyển qua xử lý lỗi chuyển đổi hiện có.
  - Timeout không tham gia chữ ký chuyển đổi; chỉ đổi timeout không tạo lại tệp đã chuyển.
  - Timeout cho từng tệp độc lập với `build.worker.conversion-timeout-seconds`.
- **Trạng thái:** Chưa chạy

### F. Thời điểm chuyển đổi và phát hiện cập nhật bản gốc

#### TC-TIMING-001 — PRECONVERT, BACKGROUND và ON_DEMAND

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Cùng một bộ dữ liệu thư viện và hồ sơ; dùng converter có thể quan sát thời điểm chạy.
- **Bước thực hiện:**
  1. Build lần lượt với `PRECONVERT`, `BACKGROUND`, `ON_DEMAND`.
  2. Với BACKGROUND, không phát sinh yêu cầu xem; sau đó gửi yêu cầu xem một tài liệu.
  3. Với ON_DEMAND, kiểm tra trước khi xem và sau lần xem đầu tiên.
- **Kết quả mong đợi:**
  - PRECONVERT chuyển đổi trước khi phát hành HTML.
  - BACKGROUND hoàn tất quét/tạo HTML/phát hành trước, sau đó chuyển đổi toàn bộ theo thứ tự và ưu tiên tài liệu được yêu cầu xem.
  - ON_DEMAND chỉ tạo task khi có yêu cầu xem.
  - Với hai phương thức sau, tác vụ xem dùng cấu hình được cố định tại thời điểm phát hành thành công.
- **Trạng thái:** Chưa chạy

#### TC-TIMING-002 — FULL, DIFF và TEMPLATE theo từng phương thức

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Đã có tài liệu hiện hành và tài liệu chưa từng được yêu cầu xem.
- **Bước thực hiện:**
  1. Chạy FULL trên PRECONVERT, BACKGROUND và ON_DEMAND.
  2. Chạy DIFF khi chỉ một tài liệu cần cập nhật.
  3. Chạy TEMPLATE.
- **Kết quả mong đợi:**
  - FULL của PRECONVERT/BACKGROUND đưa toàn bộ tài liệu vào diện chuyển đổi lại.
  - FULL của ON_DEMAND không tạo task cho tài liệu chưa được yêu cầu; khi xem mới kiểm tra output, bản gốc và cấu hình.
  - DIFF chỉ xử lý phần cần thiết.
  - TEMPLATE không quét bản gốc và không chuyển đổi lại.
- **Trạng thái:** Chưa chạy

#### TC-UPDATE-001 — Ba chế độ kiểm tra cập nhật bản gốc

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có tài liệu gốc và đầu ra hiện hữu; có cách thay metadata/content riêng biệt.
- **Bước thực hiện:**
  1. Với `METADATA`, đổi thời gian hoặc kích thước; sau đó đổi nội dung nhưng giữ cả hai metadata.
  2. Với `CHANGED_HASH`, đổi metadata nhưng giữ nội dung; sau đó đổi nội dung.
  3. Với `ALWAYS_HASH`, đổi nội dung nhưng giữ thời gian và kích thước.
  4. Quan sát hash có được tính hay không.
- **Kết quả mong đợi:**
  - METADATA mặc định: đổi thời gian hoặc kích thước thì xử lý lại; không tính hash để xác định thay đổi.
  - CHANGED_HASH: metadata đổi thì so hash; nội dung giống nhau bỏ qua chuyển đổi lại, nội dung khác thì xử lý lại.
  - ALWAYS_HASH: luôn kiểm tra toàn bộ nội dung và phát hiện thay đổi dù kích thước/thời gian không đổi.
  - METADATA/CHANGED_HASH không phát hiện thay đổi nội dung khi cả thời gian và kích thước giữ nguyên.
  - Dù không hash vẫn lấy bản gốc khi chuyển đổi/trích xuất; thiếu output hoặc đổi chữ ký cấu hình vẫn xử lý lại độc lập.
- **Trạng thái:** Chưa chạy

#### TC-POST-001 — Job sau phát hành bền vững, gộp yêu cầu trùng và an toàn phiên bản

- **Ưu tiên:** Cao
- **Tiền điều kiện:** BACKGROUND hoặc ON_DEMAND đã phát hành thành công; có thể gửi yêu cầu đồng thời và restart.
- **Bước thực hiện:**
  1. Gửi đồng thời nhiều yêu cầu xem cùng tài liệu.
  2. Thay bản gốc trong lúc chuyển đổi.
  3. Phát hành phiên bản mới hoặc xóa/loại tài liệu trước khi job cũ hoàn tất.
  4. Restart ứng dụng khi job còn dang dở.
- **Kết quả mong đợi:**
  - Job được lưu bền vững độc lập build context; yêu cầu đồng thời cùng tài liệu được gộp.
  - Chia sẻ giới hạn toàn hệ thống và engine với PRECONVERT.
  - Chuyển đổi từ bản sao gốc cố định; xác nhận tính nhất quán trước/sau và phiên bản phát hành trước khi chốt sản phẩm bất biến.
  - Không phân phối tệp chuyển đổi dang dở; job cũ không khôi phục phiên bản đã thay hoặc tài liệu đã xóa.
  - Sau restart, job chưa hoàn tất được kiểm tra lại; lỗi cùng nội dung không bị thử lại vô hạn.
- **Trạng thái:** Chưa chạy

#### TC-POST-002 — Tách biệt trích xuất nội dung, tìm kiếm và AI

- **Ưu tiên:** Trung bình
- **Tiền điều kiện:** Cấu hình BACKGROUND/ON_DEMAND; có kiểm tra trạng thái extraction/index/AI.
- **Bước thực hiện:** Tắt/để job chuyển đổi xem ở trạng thái chờ hoặc thất bại; kiểm tra pipeline trích xuất nội dung, chỉ mục tìm kiếm và AI.
- **Kết quả mong đợi:** Trích xuất nội dung, tìm kiếm và xử lý AI không bị phụ thuộc vào chuyển đổi chỉ phục vụ xem.
- **Trạng thái:** Chưa chạy

### G. Thumbnail

#### TC-THUMB-001 — Năng lực hỗ trợ và cấu hình mặc định

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Một engine có adapter thumbnail; engine khác chỉ khai báo extension ảnh nhưng không có adapter.
- **Bước thực hiện:**
  1. Đọc `thumbnailSupported` từ API cho từng engine.
  2. Thử chọn `generateThumbnail` trên từng engine.
  3. Đọc giá trị mặc định của thư viện.
- **Kết quả mong đợi:**
  - `generateThumbnail` mặc định false.
  - Chỉ engine có adapter thực sự mới báo hỗ trợ và cho phép chọn.
  - Chỉ đăng ký extension đầu ra ảnh không đủ để báo hỗ trợ.
- **Trạng thái:** Chưa chạy

#### TC-THUMB-002 — Kích thước, tỷ lệ, vị trí lưu và lỗi thumbnail

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Thumbnail bật; engine OSDC CLI có adapter; tài liệu có trang đầu theo chiều dọc/ngang; có fixture thành công và thất bại.
- **Bước thực hiện:**
  1. Chuyển đổi tài liệu thành công, kiểm tra `thumbnail.png`.
  2. Kiểm tra kích thước/tỷ lệ và vị trí file.
  3. Gây lỗi tạo ảnh riêng; lặp lại với tài liệu cùng định dạng được copy.
  4. Thử converter ảnh chậm quá 30 giây và engine timeout thấp hơn 30 giây.
- **Kết quả mong đợi:**
  - Tạo ảnh trang đầu PNG, giữ tỷ lệ, cạnh dài tối đa 480 px, không crop.
  - Thời gian tối đa 30 giây hoặc giới hạn engine nếu ngắn hơn.
  - Lưu trong cùng thư mục tài liệu đã chuyển đổi, không đặt cạnh bản gốc.
  - Lỗi thumbnail được ghi warning nhưng không làm thất bại chuyển đổi tài liệu; vẫn tạo ảnh khi copy bản gốc cùng định dạng.
- **Trạng thái:** Chưa chạy

#### TC-THUMB-003 — Cache, phát hành và chữ ký chuyển đổi

- **Ưu tiên:** Cao
- **Tiền điều kiện:** PRECONVERT; có tài liệu đã chuyển đổi và thumbnail.
- **Bước thực hiện:**
  1. Kiểm tra thumbnail được cache/copy/phát hành cùng tài liệu.
  2. Tắt rồi bật lại `generateThumbnail`.
  3. Chỉ đổi cài đặt hiển thị.
  4. Xóa tài liệu hoặc loại khỏi phát hành.
- **Kết quả mong đợi:**
  - Thumbnail là sản phẩm đi kèm được cache, copy, phát hành và xóa đúng vòng đời.
  - Bật/tắt tạo ảnh làm đổi chữ ký và đưa tài liệu hiện có vào diện cập nhật.
  - Chỉ đổi hiển thị thì không chuyển đổi lại; áp dụng qua tạo lại HTML.
  - Không giữ ảnh mồ côi sau khi xóa/loại tài liệu.
- **Trạng thái:** Chưa chạy

#### TC-THUMB-004 — Hàng đợi `sequentialThumbnails` của ON_DEMAND

- **Ưu tiên:** Cao
- **Tiền điều kiện:** ON_DEMAND, tạo thumbnail bật; có nhiều tài liệu; có thể theo dõi ưu tiên và restart.
- **Dữ liệu/cấu hình:** `sequentialThumbnails=false` và `true`; normalConversionTarget bất kỳ hợp lệ.
- **Bước thực hiện:**
  1. Xác nhận trường chỉ áp dụng trong ON_DEMAND khi `generateThumbnail=true`.
  2. Bật tuần tự; phát hành thành công và quan sát queue.
  3. Đồng thời tạo yêu cầu xem và chuyển đổi thông thường.
  4. Restart giữa các thumbnail; gây một lần tạo ảnh thất bại và một lần gián đoạn lặp.
- **Kết quả mong đợi:**
  - Tùy chọn mặc định false; hàng đợi ảnh chỉ được tạo sau phát hành thành công.
  - Queue chỉ chứa ảnh, ưu tiên thấp hơn yêu cầu xem/chuyển đổi thông thường, dùng slot trống toàn hệ thống và engine.
  - Không tính vào `normalConversionTarget`; không ngắt tác vụ đang chạy.
  - Không tạo yêu cầu xem hoặc job chuyển đổi nội dung tài liệu.
  - Restart giữ số lần thử và khôi phục phần gián đoạn; không retry vô hạn khi lỗi/gián đoạn lặp lại.
- **Trạng thái:** Chưa chạy

#### TC-THUMB-005 — Không dùng thumbnail cũ sau khi bản gốc/cấu hình thay đổi

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có thumbnail đã tạo; có thể trì hoãn job để thay đổi trạng thái trước khi hoàn tất.
- **Bước thực hiện:**
  1. Bắt đầu tạo ảnh rồi đổi bản gốc/path/metadata/size/content hoặc chữ ký chuyển đổi.
  2. Lặp lại khi tài liệu bị loại khỏi phát hành, bị xóa hoặc `generateThumbnail` bị tắt.
  3. Tạo ảnh hoàn tất trước rồi phát sinh yêu cầu xem.
- **Kết quả mong đợi:**
  - Nhận dạng phiên bản ảnh dựa trên kết nối/path, thời gian, kích thước, hash nếu có và chữ ký.
  - Khi đổi bản gốc, không nhận ảnh cũ; tạo phiên bản mới.
  - Job cũ không khôi phục ảnh sau xóa/loại/tắt tùy chọn.
  - Ảnh đã hoàn tất có thể được tái sử dụng làm sản phẩm tài liệu khi xem.
- **Trạng thái:** Chưa chạy

### H. Nhật ký chuyển đổi

#### TC-LOG-001 — Tạo log cho PRECONVERT và chuyển đổi sau phát hành

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Chạy ít nhất một PRECONVERT và một BACKGROUND/ON_DEMAND.
- **Bước thực hiện:**
  1. Chuyển đổi thành công và thất bại.
  2. Kiểm tra đường dẫn và encoding file log.
  3. Thử retry, skip và phát hành lại.
- **Kết quả mong đợi:**
  - PRECONVERT dùng `build:<jobId>`; BACKGROUND/ON_DEMAND dùng `preview:<libraryId>:<fileId>`.
  - Log nằm dưới `library_<id>/logs/conversions/`, encoding UTF-8.
  - PRECONVERT ghi theo job; sau phát hành ghi theo tài liệu.
  - Trial number, publication generation và artifact revision phân biệt từng lần chạy; retry/republication không ghi đè lịch sử.
- **Trạng thái:** Chưa chạy

#### TC-LOG-002 — Nội dung log, argv và che bí mật

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Converter fixture ghi stdout/stderr; cấu hình có secret test; chạy ca thành công, lỗi trước khởi chạy và lỗi chuẩn bị input.
- **Bước thực hiện:** Thực thi các ca và tải log; kiểm tra command, argv, đầu ra và trường thông tin.
- **Kết quả mong đợi:**
  - Ghi engine/profile ID, outputFormat, tham số hiệu lực, file đích, argv JSON giữ ranh giới đối số, timeout, working directory, PID, output stdout/stderr gộp, exit code, thời lượng, exception và nguyên nhân retry/skip/discard.
  - Secret được che trước khi serialize command thành JSON và trước khi lưu output.
  - Lưu process output cả khi thành công.
  - Lỗi trước khi chạy/chuẩn bị input ghi mọi thông tin sẵn có.
  - Lỗi ghi log không làm đổi kết quả chuyển đổi tài liệu.
- **Trạng thái:** Chưa chạy

#### TC-LOG-003 — API đọc log: quyền, giới hạn cuối tệp và download

- **Ưu tiên:** Cao
- **Tiền điều kiện:** Có log lớn hơn 512 KiB; có tài khoản được phép và không được phép xem thư viện.
- **Bước thực hiện:**
  1. Gọi `GET /api/build/convert-jobs/log?libraryId=<id>&taskKey=<key>` với quyền hợp lệ.
  2. Gọi với library khác, taskKey không thuộc library, không đăng nhập và thiếu quyền.
  3. Gọi thêm `download=true`.
  4. Kiểm tra response headers và điểm bắt đầu của phần log trả về.
- **Kết quả mong đợi:**
  - Chỉ trả log sau khi xác thực quyền xem thư viện và task thuộc thư viện.
  - Đọc mặc định không quá 512 KiB cuối tệp, cắt tại ranh giới dòng.
  - Thông báo rõ phần đầu log đã bị lược bỏ.
  - `download=true` tải toàn bộ file; mọi response có `Cache-Control: no-store`.
  - Không tiết lộ log của thư viện khác trong lỗi hoặc response.
- **Trạng thái:** Chưa chạy

#### TC-LOG-004 — UI xem log, polling và giữ trạng thái danh sách

- **Ưu tiên:** Trung bình
- **Tiền điều kiện:** Có log đang được ghi; UI danh sách có phân trang và bộ lọc.
- **Bước thực hiện:**
  1. Đặt bộ lọc và chuyển đến trang khác.
  2. Mở “Hiển thị nhật ký thực thi”; đợi tối thiểu hai chu kỳ polling.
  3. Đóng viewer và tải log.
  4. Kiểm tra danh sách có log rất dài.
- **Kết quả mong đợi:**
  - Danh sách hiển thị dòng lỗi đầu tiên với chiều rộng giới hạn.
  - Viewer dùng chung trình xem hiện có và lấy phần cuối mỗi 3 giây khi đang mở.
  - Khi đóng viewer, bộ lọc, trang và vị trí nền được giữ nguyên.
  - UI tuân theo giới hạn số dòng; thao tác xem và tải log hoạt động riêng.
- **Trạng thái:** Chưa chạy

#### TC-LOG-005 — Lượt chạy cũ chưa có log chi tiết

- **Ưu tiên:** Trung bình
- **Tiền điều kiện:** Có record lượt chạy cũ không lưu log chi tiết.
- **Bước thực hiện:** Mở log và tải xuống; thay đổi cấu hình hiện tại rồi xem lại.
- **Kết quả mong đợi:**
  - Hiển thị trạng thái, thời điểm và lỗi đã lưu; thông báo rõ không có log chi tiết.
  - Không suy đoán/tái tạo command lịch sử từ cấu hình hiện tại.
- **Trạng thái:** Chưa chạy

## 4. Bản ghi thực thi

Tester sao chép bảng này cho mỗi lượt chạy:

| Trường | Giá trị |
| --- | --- |
| Ngày/giờ và môi trường | Chưa ghi |
| Phiên bản ứng dụng / OSDC | Chưa ghi |
| DB và schema mode | Chưa ghi |
| Cấu hình global concurrency | Chưa ghi |
| Công cụ/hồ sơ được dùng | Chưa ghi |
| Test case ID | Chưa ghi |
| Kết quả | Chưa chạy |
| Bằng chứng (test report, API, log, DB) | Chưa ghi |
| Lỗi/issue liên quan | Chưa ghi |
| Người chạy | Chưa ghi |
