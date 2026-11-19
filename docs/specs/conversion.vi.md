# Hồ sơ công cụ chuyển đổi

Tham khảo [mục lục](../README.md) để biết tiêu chí xác nhận.

## Trách nhiệm của cấu hình

| Cấu hình | Trách nhiệm |
| --- | --- |
| Công cụ chuyển đổi | Cách thực thi, đích kết nối, định dạng đầu vào/đầu ra được hỗ trợ, giới hạn số tiến trình chạy đồng thời và thời gian chờ chuyển đổi tệp |
| Hồ sơ chuyển đổi | Công cụ được sử dụng, một `outputFormat`, bộ tham số mặc định và ghi đè thời gian chờ |
| Thư viện | Hồ sơ được sử dụng, `parameterOverrides`, tham số bổ sung, trình xem và phiên bản cố định |

Định dạng đầu ra được xác định trong hồ sơ. Không sử dụng thiết kế cũ cho phép chọn riêng định dạng đầu ra trong thư viện. Định dạng được lưu phải nằm trong phạm vi định dạng mà công cụ hỗ trợ; xác nhận tính tương thích với trình xem của thư viện khi lưu và khi build.

Tham số hiệu lực là kết quả áp dụng các ghi đè và phần bổ sung của thư viện lên giá trị mặc định trong hồ sơ. Không trộn danh sách tham số riêng của công cụ vào đó. Xác thực các tham số bắt buộc, chỉ đọc, giá trị boolean, chuỗi rỗng và xung đột đối số. Đưa định dạng đầu ra, cấu hình thực thi và tham số hiệu lực vào chữ ký dùng để xác định thay đổi; cố định cấu hình tại thời điểm bắt đầu build. Thay đổi định dạng hoặc giá trị hiệu lực sẽ cần chuyển đổi lại; lưu lại cùng cấu hình hoặc chỉ đổi trình xem nhưng giữ nguyên định dạng thì không cần chuyển đổi lại.

Tiêu đề trong giao diện là “Cấu hình tham số”. Trường có thể chỉnh sửa có nền trắng, trường chỉ đọc có nền xám; mục đã điều chỉnh có viền xanh và thao tác đặt lại.

## Chuyển PDF gốc và chuyển đổi định dạng đầu ra (#6837)

Thêm PDF vào các định dạng đầu vào được hỗ trợ của công cụ OSDC tiêu chuẩn; áp dụng khi đăng ký mới và khi cập nhật lúc khởi động các công cụ tiêu chuẩn hiện có. Nếu định dạng đầu ra là SVG, chuyển PDF gốc thành SVG theo từng trang bằng OSDC, tạo thông tin tài liệu SVG rồi chuyển cho trình xem được chỉ định. Chỉ áp dụng cài đặt “bỏ qua chuyển đổi khi định dạng giống nhau” khi bản gốc và đầu ra cùng định dạng. Không xác định định dạng đầu ra chỉ bằng cách thay đổi trình xem.

Chuyển từ đầu ra PDF sang SVG luôn cần chuyển đổi lại, bất kể bản gốc có được cập nhật hay không. Với tài liệu từng bị bỏ qua do “phần mở rộng không thuộc đối tượng chuyển đổi”, hãy thử xử lý lại trong bản build sai khác nếu cấu hình đầu vào/đầu ra hiện tại cho phép. Không khôi phục tài liệu đã bị xóa, bị loại khỏi bản phát hành hoặc vẫn dùng định dạng không được hỗ trợ. Chuyển đổi trước, chuyển đổi nền và chuyển đổi theo yêu cầu phải dùng cùng một quy tắc xác định định dạng được hỗ trợ; chuyển đổi sau khi phát hành được áp dụng bằng cách phát hành lại với cấu hình cập nhật.

Nếu công cụ không thể chuyển đổi PDF do mã hóa, hạn chế quyền hoặc nguyên nhân tương tự, hãy ghi nhận lý do như một lỗi chuyển đổi thông thường. Tài liệu đi kèm OSDC V12 “Chuyển đổi từ PDF” quy định hỗ trợ PDF→SVG và các giới hạn như PDF cấm sao chép hoặc trích xuất nội dung.

## Đường dẫn thực thi khi đăng ký mới

Khi đăng ký công cụ chuyển đổi mới, hiển thị đường dẫn thực thi để trống. `convertUrl` trong `GET /api/converter-engines/defaults` phải là chuỗi rỗng. Placeholder minh họa lệnh Windows giả định `C:\Tools\SampleConverter.exe -in {input} -out {output}`; không lưu placeholder làm giá trị nhập. Đường dẫn thực thi vẫn là bắt buộc; người dùng phải nhập vị trí cài đặt và đối số thực tế. Khi chỉnh sửa, hiển thị giá trị đã lưu. Không thay đổi giá trị mặc định theo hệ điều hành dùng để khởi tạo công cụ tiêu chuẩn lúc khởi động, cũng như các lệnh hiện có đã đăng ký.

## Giới hạn số tiến trình chạy đồng thời

`maxConcurrentJobs` của công cụ là số nguyên từ 1 đến 64, mặc định là 1. Khi tạo, nếu bỏ qua thì dùng 1; khi cập nhật, nếu bỏ qua hoặc là null thì giữ nguyên giá trị hiện tại. Hiển thị trường này trong màn hình chỉnh sửa, danh sách, API chi tiết và bản xuất JSON. Áp dụng giá trị mặc định 1 cho cột DB của các công cụ hiện có.

Tất cả thư viện dùng cùng ID công cụ chia sẻ chung các lượt chạy. Chỉ khởi chạy khi đồng thời thỏa mãn giới hạn này và giới hạn toàn hệ thống `build.converter.concurrency` (mặc định là 4).

Quản lý giới hạn toàn hệ thống trong mục cài đặt hệ thống “Danh sách cài đặt nâng cao → Số worker chuyển đổi toàn hệ thống” và lưu số nguyên từ 1 đến 64. Sau khi lưu, áp dụng vào kích thước worker pool và số lượt chạy có thể khởi động ở lần kiểm tra xử lý kế tiếp (thường mỗi 1 giây); không cần khởi động lại. Khi giảm giới hạn, các lượt chuyển đổi đang chạy vẫn tiếp tục đến khi hoàn tất. Chỉ lần đầu mới lấy giá trị cài đặt khởi động để ghi vào DB; sau đó ưu tiên cài đặt DB hợp lệ. Nếu cài đặt bị vô hiệu hóa, bị xóa hoặc có giá trị hiện có không hợp lệ, quay về cài đặt khởi động; nếu không lấy được DB thì chờ, không khởi chạy lượt mới.

Đặt chỗ chạy trước khi khởi động; số chỗ đang đặt cũng được tính vào giới hạn. Trong lúc chờ chỗ, tác vụ giữ trạng thái `QUEUED`, không thay đổi thời điểm bắt đầu hoặc số lần thử. Trong mỗi thư viện, xét các tác vụ theo thứ tự ưu tiên rồi đến thời điểm tạo; bỏ qua công cụ đã hết chỗ và tiếp tục xử lý bằng công cụ hoặc thư viện khác. Các chỗ trống được bổ sung qua lần kiểm tra mỗi giây của bộ lập lịch chung.

### Phân bổ giữa các thư viện

Bộ lập lịch chung xét xuyên suốt hàng đợi chuyển đổi trước và chuyển đổi sau khi phát hành, rồi phân bổ từng chỗ trống tiếp theo. Ưu tiên cao nhất là yêu cầu xem theo yêu cầu và các tác vụ chuyển đổi nền được tăng ưu tiên do có yêu cầu xem. Không ngắt các lượt chuyển đổi đang chạy.

Cài đặt thư viện “Chuyển đổi・Build → Số lượt chuyển đổi thông thường cần ưu tiên” (`converterConfig.normalConversionTarget`) nhận giá trị từ 1 đến 64, mặc định là 1. Đây là mục tiêu trong lúc bận đối với chuyển đổi trước và chuyển đổi nền thông thường, không phải mức tối thiểu được đảm bảo hay giới hạn tối đa. Ẩn mục này khi chọn phương thức chuyển đổi theo yêu cầu. Khi đổi phương thức, giữ nguyên giá trị đang nhập và giá trị đã lưu; hiển thị lại khi quay về chuyển đổi trước hoặc nền. Áp dụng từ lần phân bổ sau khi lưu, không cần build lại, khởi động lại hay chuyển đổi lại. Không đưa giá trị này vào xác định thế hệ của chỉ mục nội dung hoặc dữ liệu đầu vào AI; giữ nguyên các tác vụ chỉ mục và kết quả đã xử lý.

Ưu tiên thư viện chưa đạt mục tiêu chuyển đổi thông thường; phân bổ theo thứ tự tỷ lệ sử dụng mục tiêu thấp nhất (số tác vụ đang chạy / mục tiêu), nếu bằng nhau thì thư viện được phân bổ lâu nhất trước. Bên trong mỗi thư viện, xử lý theo thứ tự tiếp nhận. Nếu mọi thư viện đã đạt mục tiêu, phân bổ chỗ dư theo thứ tự tiếp nhận chung của hai hàng đợi chuyển đổi trước và nền, với cùng mức ưu tiên. Bỏ qua lượt chờ chỗ của công cụ không thể chạy và chuyển sang tác vụ khả thi khác.

Khi không có tranh chấp, ngay cả một thư viện cũng có thể sử dụng toàn bộ chỗ trống. Nếu sau đó thư viện khác bắt đầu chờ, hãy xem xét lại phân bổ từ một chỗ mỗi khi một lượt đang chạy kết thúc. Ví dụ, nếu A, B, C mỗi thư viện cần 2 lượt trong khi tổng giới hạn là 4, không vượt quá giới hạn toàn hệ thống; có thể chạy phân bổ 2, 1, 1 và luân phiên công bằng khi các lượt hoàn tất. Yêu cầu xem được ưu tiên hơn mục tiêu, vì vậy chuyển đổi thông thường có thể tạm thời thấp hơn mục tiêu; sau khi xử lý yêu cầu xem xong, bổ sung phần còn thiếu. Không dành chỗ cố định cho build và không đặt thời gian chờ khi chỉ có một worker. Giới hạn toàn hệ thống và giới hạn của công cụ áp dụng cho mọi phương thức.

Khi quyết định khởi chạy, dùng giới hạn mới nhất, không dùng snapshot tại thời điểm bắt đầu build. Khi giảm giới hạn, không dừng tác vụ đang chạy; chờ đến khi số tác vụ đang chạy thấp hơn giới hạn mới mới khởi chạy thêm. Giải phóng chỗ khi thành công, thất bại, hết thời gian, bị hủy hoặc không thể đưa vào hàng đợi. Với tác vụ đang chạy, không giải phóng chỉ vì nhận thông báo hủy Future; chờ cho tác vụ thực sự kết thúc. Xử lý các tác vụ chưa hoàn tất khi khởi động lại giữ nguyên như hiện tại.

Quản lý chỗ chạy trong ứng dụng. Tương tự nền tảng thực thi build hiện tại, không hỗ trợ nhiều instance ứng dụng cùng chạy đồng thời khi chia sẻ một DB. Số lượt chạy đồng thời không được đưa vào kết quả chuyển đổi hay chữ ký xác định khác biệt.

Với `ddl-auto=update`, cột được thêm tự động. Trong môi trường `validate`, hãy dừng ứng dụng và sao lưu, rồi chạy một lần lệnh sau trên DB quản lý trước khi khởi động phiên bản mới (H2).

```sql
ALTER TABLE converter_engine ADD COLUMN IF NOT EXISTS max_concurrent_jobs INTEGER DEFAULT 1 NOT NULL;
```

Khởi đầu OSDC với giá trị 1; đo CPU, bộ nhớ và thời gian chuyển đổi bằng các tài liệu thực tế tiêu biểu, sau đó tăng dần lên 2, 3. Nếu đăng ký cùng một chương trình thực thi dưới các ID công cụ khác nhau, mỗi ID có giới hạn riêng, vì vậy cần cân nhắc tổng tải.

## Thời gian chờ chuyển đổi tệp

`timeoutSeconds` của công cụ là số nguyên từ 1 đến 2147483647 (giây), mặc định 300 giây. Khi tạo công cụ, nếu bỏ qua hoặc là null thì dùng 300; khi cập nhật, nếu bỏ qua hoặc là null thì giữ nguyên giá trị hiện tại. `timeoutSeconds` của hồ sơ là số nguyên trong cùng phạm vi hoặc null. Khi tạo/cập nhật hồ sơ, nếu bỏ qua hoặc là null thì trở về kế thừa cấu hình của công cụ. Trong màn hình chỉnh sửa hồ sơ, hiển thị ô trống là “Kế thừa cài đặt công cụ chuyển đổi” và thông báo giá trị hiện tại của công cụ. API phải từ chối số thập phân, chuỗi, 0, số âm và số ngoài phạm vi số nguyên.

Thứ tự áp dụng: giá trị trong hồ sơ, giá trị trong công cụ, mặc định 300 giây. Cho phép cấu hình khi đăng ký/chỉnh sửa công cụ và hồ sơ; lưu giá trị trong API chi tiết/danh sách và bản xuất JSON của công cụ. Không sử dụng cấu hình áp dụng đồng loạt trước đây `build.converter.timeout-seconds`. Khi cài đặt vào môi trường hiện có, thời gian chờ là 300 giây; nếu trước đây đã điều chỉnh bằng cấu hình khởi động, cần nhập lại giá trị đó cho công cụ qua giao diện.

Lấy giá trị mới nhất từ DB quản lý ngay trước khi bắt đầu chuyển đổi bên ngoài cho từng tệp. Áp dụng không cần khởi động lại cho các lượt chuyển đổi hoặc thử lại bắt đầu sau khi lưu; không thay đổi giới hạn thời gian của tiến trình đang chạy. Giữ snapshot cài đặt đầu ra tại thời điểm bắt đầu build, nhưng đọc giá trị thời gian chờ mới nhất. Nếu công cụ đang dùng trong hồ sơ thay đổi giữa chừng, chỉ sử dụng giá trị của hồ sơ khớp với công cụ đã được chọn cho build. Khi không lấy được DB hoặc cấu hình không hợp lệ, không chạy mà không có giới hạn; chuyển sang xử lý lỗi chuyển đổi hiện có. Không đưa thời gian chờ vào chữ ký kết quả chuyển đổi; chỉ thay đổi giá trị này không làm chuyển đổi lại tệp đã xử lý. Giá trị này độc lập với giới hạn chờ toàn build `build.worker.conversion-timeout-seconds`.

`ddl-auto=update` tự tạo các cột bổ sung. Trong môi trường `validate`, hãy dừng ứng dụng và sao lưu, rồi chạy các lệnh sau trên DB quản lý trước khi khởi động phiên bản mới (H2; giữ nguyên các hàng hiện có và hỗ trợ chạy lại). Các công cụ hiện có nhận 300 giây; hồ sơ hiện có kế thừa cấu hình.

```sql
ALTER TABLE converter_engine ADD COLUMN IF NOT EXISTS timeout_seconds INTEGER DEFAULT 300;
ALTER TABLE conversion_profile ADD COLUMN IF NOT EXISTS timeout_seconds INTEGER;
```

## Hồ sơ PDF và SVG tiêu chuẩn

Giữ đồng thời hồ sơ PDF và SVG tiêu chuẩn, đồng thời duy trì PDF là lựa chọn mặc định. Thêm hồ sơ SVG tiêu chuẩn cho công cụ dùng phương thức CLI và hỗ trợ SVG. Giữ nguyên hồ sơ trùng tên hiện có hoặc đã bị vô hiệu hóa; không tạo trùng hoặc ghi đè mỗi lần khởi động.

Tạo SVG theo từng trang, ghi tổng số trang và kích thước vào manifest. Cấu hình tiêu chuẩn gồm 8 mục, chẳng hạn SVG 1.1, nhúng ảnh, tự động chuyển đổi ảnh và chất lượng JPEG 80. Mức 80 là giá trị chuẩn do Astack chỉ định. Nền tảng chuyển đổi quản lý việc chia trang và tên tệp; cố định việc nhúng ảnh để dùng ngoại tuyến.

| Tham số | Giá trị ban đầu | Nội dung |
| --- | --- | --- |
| svgver | SVG1.1 | Đặc tả SVG |
| svgip | 0 (chỉ đọc) | Nhúng tất cả hình ảnh |
| svgic | 0 | Tự động chọn định dạng hình ảnh |
| svgjq | 80 | Chất lượng JPEG |
| docpagenumbercorrection | false | Điều chỉnh số trang trong mục lục |
| xlscontinuepagenumber | false | Đánh số trang Excel liên tục |
| pptnotes | false | Ghi chú PowerPoint |
| extlevel | 2 | Mức dừng do lỗi |

Sao chép đầu vào vào vùng làm việc; tách biệt vùng lưu trữ không công khai với nơi phát hành sản phẩm. Duy trì liên kết giữa snapshot đầu vào và phiên bản phát hành thành công. Nhận quyền hoàn tất thư mục bằng cập nhật trạng thái có điều kiện trong DB để không tạo trùng từ các thông báo đồng thời.

## Bỏ qua khi định dạng đầu ra trùng với bản gốc

Nếu phần mở rộng của bản gốc trùng với định dạng đầu ra và bật tùy chọn bỏ qua chuyển đổi, dùng bản sao của tệp gốc và đặt tác vụ chuyển đổi thành `SKIPPED`. Không đơn thuần đổi trạng thái thành thành công. Duy trì việc đăng ký tìm kiếm và dữ liệu đầu vào AI cho PDF đã sao chép; lỗi sao chép hoặc đăng ký chỉ mục phải chuyển sang xử lý lỗi. Xem [Tác vụ・Phát hành](jobs-and-publication.md) về việc phát hành sau lỗi riêng lẻ.

## API quản trị và căn cứ

Quản lý qua `/api/converter-engines` và `/api/conversion-profiles` dành cho quản trị viên. Hồ sơ có các API liệt kê, chi tiết, danh sách đang hoạt động, danh sách theo công cụ, tạo, cập nhật và xóa.

- [ConversionProfile](../../backend/src/main/java/jp/astack/backend/model/entity/ConversionProfile.java), [ConversionProfileService](../../backend/src/main/java/jp/astack/backend/service/ConversionProfileService.java)
- [EffectiveConversionSettings](../../backend/src/main/java/jp/astack/backend/build/pipeline/converter/EffectiveConversionSettings.java), [ConversionBuildSnapshotService](../../backend/src/main/java/jp/astack/backend/build/pipeline/converter/ConversionBuildSnapshotService.java)
- [ConverterService](../../backend/src/main/java/jp/astack/backend/build/pipeline/converter/ConverterService.java), [FolderCompletionChecker](../../backend/src/main/java/jp/astack/backend/build/pipeline/generator/FolderCompletionChecker.java)
- [DevDataInitializer](../../backend/src/main/java/jp/astack/backend/config/DevDataInitializer.java), [Kiểm thử API hồ sơ](../../backend/src/test/java/jp/astack/backend/controller/ConversionProfileOutputApiTest.java)

## Hỗ trợ tệp trùng tên và bộ nhớ đệm chuyển đổi (chưa cung cấp)

Astack4 chưa cung cấp cài đặt “Hỗ trợ tệp trùng tên” để kiểm soát tài liệu trùng tên nhưng khác phần mở rộng, cũng như cài đặt bộ nhớ đệm chuyển đổi để người dùng bật/tắt ([#6726](https://developer.astack.jp/redmine/issues/6726)). Đây là hai khái niệm khác nhau; không diễn giải hỗ trợ tệp trùng tên thành bộ nhớ đệm chuyển đổi. Do đặc tả chưa được xác định, không hiển thị các lựa chọn trên giao diện. Phát hiện sai khác hiện có và bỏ qua chuyển đổi vẫn tuân theo quy tắc mặc định tương ứng.

Duy trì đọc/ghi `converterConfig.enableConvertCache` để tương thích với API, JSON hiện có và chức năng nhập, nhưng không thay đổi xử lý thực thi. Giữ khóa này trong biểu mẫu; bỏ ánh xạ khi gửi từ tên UI cũ `enableSameNameFile`. Duy trì giá trị hiện có khi chỉnh sửa, sao chép và lưu lại; không xóa hàng loạt hoặc di chuyển dữ liệu. Không khôi phục ý định xử lý tệp trùng tên từ các giá trị đã lưu bằng UI cũ.

Việc loại bỏ mục hỗ trợ tệp trùng tên chỉ là biện pháp tạm thời; dự kiến sẽ bổ sung chức năng trong tương lai. #6726 đã được mở lại và không đóng cho đến khi hoàn tất xác định đặc tả, triển khai và kiểm thử xử lý tệp trùng tên. Không xem việc xóa ánh xạ sai hoặc ẩn mục này là đã hoàn thành triển khai chức năng.

## Thời điểm chuyển đổi và kiểm tra cập nhật bản gốc (#6794)

`converterConfig.conversionTiming` của thư viện gồm `PRECONVERT` (chuyển đổi trước, mặc định), `BACKGROUND` (chuyển đổi nền) và `ON_DEMAND` (chuyển đổi theo yêu cầu). Với chuyển đổi trước, vẫn phát hành HTML sau khi chuyển đổi như hiện tại. Hai phương thức còn lại hoàn tất quét, tạo HTML và phát hành trước; sau khi phát hành thành công, thực hiện chuyển đổi riêng cho việc xem bằng cấu hình đã cố định. BACKGROUND xử lý tuần tự toàn bộ tài liệu và ưu tiên tài liệu có yêu cầu xem. ON_DEMAND chỉ tạo tác vụ chuyển đổi khi có yêu cầu xem. FULL của chuyển đổi trước và nền đưa mọi tài liệu trở lại diện cần chuyển đổi. Với ON_DEMAND, ngay cả khi FULL cũng không tạo tác vụ cho tài liệu chưa được yêu cầu; khi xem mới kiểm tra sự tồn tại của sản phẩm, bản gốc và cấu hình. DIFF chỉ xử lý phần cần thiết; TEMPLATE không quét bản gốc hoặc chuyển đổi lại.

`converterConfig.updateCheckMode` có ba lựa chọn sau. Sử dụng khi build và khi xem qua phương thức tích hợp.

| Giá trị | Nhãn giao diện | Cách hoạt động |
| --- | --- | --- |
| `METADATA` (mặc định) | Kiểm tra ngày cập nhật và kích thước | Xử lý lại nếu một trong hai thay đổi. Không tính hash để xác định thay đổi |
| `CHANGED_HASH` | Kiểm tra nội dung khi phát hiện thay đổi | Khi metadata thay đổi, so sánh hash nội dung; nếu nội dung giống nhau thì bỏ qua chuyển đổi lại |
| `ALWAYS_HASH` | Luôn kiểm tra nội dung | Kiểm tra toàn bộ nội dung ngay cả khi metadata không đổi. Phát hiện cả thay đổi có cùng kích thước và thời gian |

METADATA/CHANGED_HASH không phát hiện được thay đổi nội dung nếu thời gian và kích thước không đổi. Ngay cả khi không dùng hash, vẫn cần lấy bản gốc khi chuyển đổi hoặc trích xuất nội dung. Thiếu đầu ra và thay đổi chữ ký cài đặt chuyển đổi đều thuộc diện xử lý lại, độc lập với việc bản gốc có thay đổi hay không.

Lưu bền vững các tác vụ chuyển đổi sau phát hành, độc lập với ngữ cảnh build, và gộp các yêu cầu đồng thời cho cùng tài liệu. Chia sẻ giới hạn lượt chuyển đổi toàn hệ thống và theo công cụ với chuyển đổi trước. Cố định bản sao gốc đã lấy về để chuyển đổi; chỉ xác nhận sản phẩm bất biến sau khi kiểm tra tính nhất quán trước/sau khi lấy và phiên bản phát hành. Không phân phối tệp đang được chuyển đổi. Không để tác vụ cũ khôi phục phiên bản đã phát hành lại hoặc tài liệu đã xóa. Khi khởi động lại, kiểm tra lại các tác vụ chưa hoàn tất và không thử lại vô hạn khi cùng nội dung liên tục thất bại.

Trích xuất nội dung, chỉ mục tìm kiếm và xử lý AI độc lập với chuyển đổi phục vụ việc xem. Tham khảo [Trích xuất nội dung bản gốc](source-text-extraction.md).

## Ảnh thu nhỏ tài liệu (#6838)

`converterConfig.generateThumbnail` mặc định là false. Chỉ có thể chọn nếu công cụ có adapter tương ứng. Hỗ trợ ban đầu là kết nối dòng lệnh của OSDC; chỉ đăng ký phần mở rộng đầu ra hình ảnh không được xem là có năng lực hỗ trợ. API công cụ trả về `thumbnailSupported`.

Xuất trang đầu tiên thành PNG cùng thời điểm và dùng chỗ chạy như chuyển đổi tài liệu; chuẩn hóa để cạnh dài không quá 480 px và giữ nguyên tỷ lệ. Không cắt ảnh. Chuyển đổi ảnh tối đa 30 giây (hoặc theo giới hạn thấp hơn của công cụ). Ghi nhận lỗi tạo ảnh dưới dạng cảnh báo, không làm thay đổi kết quả thành công của chuyển đổi tài liệu. Vẫn tạo ảnh khi sao chép bản gốc cùng định dạng.

Lưu `thumbnail.png` trong cùng thư mục dành cho tài liệu đã chuyển đổi. Với chuyển đổi trước, lưu vào bộ nhớ đệm, sao chép, phát hành và xóa như sản phẩm đi kèm. Không ghi cạnh bản gốc. Bật tạo ảnh sẽ được đưa vào chữ ký chuyển đổi và làm các tài liệu hiện có thuộc diện cập nhật. Chỉ thay đổi cài đặt hiển thị thì không chuyển đổi lại; áp dụng bằng cách tạo lại HTML.

Với ON_DEMAND, cung cấp tùy chọn bổ sung `sequentialThumbnails` (nhãn giao diện “Tạo ảnh thu nhỏ tuần tự”, mặc định false). Chỉ áp dụng khi đã bật tạo ảnh và dùng ON_DEMAND. Sau khi phát hành thành công, tạo hàng đợi bền vững chỉ chứa tác vụ hình ảnh; ưu tiên thấp hơn yêu cầu xem và chuyển đổi thông thường, sử dụng chỗ trống toàn hệ thống và của công cụ. Không tính vào số lượt chuyển đổi thông thường cần ưu tiên và không ngắt tác vụ đang chạy. Không tạo yêu cầu xem tài liệu hoặc tác vụ chuyển đổi nội dung tài liệu. Khi khởi động lại, giữ số lần thử và khôi phục phần bị gián đoạn; không tự động thử lại vô hạn khi liên tục bị gián đoạn hoặc tạo ảnh thất bại.

Định danh phiên bản ảnh bằng kết nối và đường dẫn bản gốc, thời gian, kích thước, hash (nếu đã có) và chữ ký chuyển đổi. Khi có thay đổi, không dùng ảnh cũ mà tạo ảnh mới. Không để xử lý cũ khôi phục ảnh sau khi tài liệu bị loại khỏi bản phát hành, bị xóa hoặc tùy chọn bị tắt. Có thể tái sử dụng ảnh hoàn tất trước đó làm sản phẩm tài liệu khi xem.

## Nhật ký thực thi chuyển đổi tệp (#6842)

Danh sách hiển thị dòng đầu tiên của lỗi với độ rộng giới hạn; xem toàn bộ qua các thao tác “Hiển thị nhật ký thực thi” và “Tải nhật ký xuống” trong cột thao tác. Dùng chung trình xem nhật ký hiện có; khi mở, lấy phần cuối nhật ký mỗi 3 giây. Giữ nguyên danh sách, bộ lọc và vị trí trang ở nền.

Dùng `build:<jobId>` cho chuyển đổi trước và `preview:<libraryId>:<fileId>` cho chuyển đổi nền/theo yêu cầu; lưu tệp nhật ký UTF-8 trong vùng dữ liệu của thư viện tại `library_<id>/logs/conversions/`. Chuyển đổi trước ghi theo từng tác vụ; chuyển đổi sau phát hành ghi theo từng tài liệu. Phân biệt lần chạy bằng số lần thử, thế hệ phát hành và revision sản phẩm. Không ghi đè lịch sử của các lần thử lại hoặc phát hành lại.

Ghi lại công cụ và ID hồ sơ tại thời điểm thực thi, định dạng đầu ra, tham số đã xác định, tệp đích, lệnh thực tế sau khi mở rộng và hợp nhất tham số (argv dưới dạng JSON, giữ nguyên ranh giới đối số), thời gian chờ, thư mục làm việc, PID, đầu ra gộp stdout/stderr, mã thoát và thời gian xử lý, ngoại lệ, cũng như việc loại bỏ do thử lại, bỏ qua hoặc thay đổi phiên bản phát hành. Che giá trị bí mật trước khi chuyển lệnh thành JSON và khi lưu đầu ra. Lưu đầu ra tiến trình kể cả khi thành công. Với lỗi trước khi khởi chạy hoặc trong lúc chuẩn bị đầu vào, ghi lại những thông tin có thể thu thập. Lỗi lưu nhật ký không làm thay đổi kết quả chuyển đổi tài liệu.

`GET /api/build/convert-jobs/log?libraryId=<id>&taskKey=<key>` kiểm tra quyền xem thư viện và sự tồn tại của tác vụ thuộc thư viện, sau đó trả về tối đa 512 KiB ở cuối tệp, theo ranh giới dòng. Thông báo rõ phần đầu đã bị lược bỏ; `download=true` phân phối toàn bộ tệp. Phản hồi có `Cache-Control: no-store`. UI cũng áp dụng giới hạn số dòng hiện có. Với các lượt chạy cũ không có nhật ký, hiển thị trạng thái, thời gian và lỗi đã lưu, đồng thời nêu rõ không có nhật ký chi tiết. Không suy đoán và tạo lệnh quá khứ từ cấu hình hiện tại.
