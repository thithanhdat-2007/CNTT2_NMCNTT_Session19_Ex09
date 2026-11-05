# BÁO CÁO AN NINH MẬT KHẨU VÀ THIẾT KẾ CHÍNH SÁCH QUẢN TRỊ CHO PIXELFORGE STUDIO

## PHẦN 1: BÁO CÁO PHÂN TÍCH KIẾN TRÚC AN NINH MẬT KHẨU

### Câu 1: Phân tích mức độ nguy hiểm khi lưu mật khẩu trên trình duyệt Google Chrome (tệp Login Data) trước mã độc Infostealer

Trong quá trình làm việc, nhiều lập trình viên thường chọn tính năng "Lưu mật khẩu" mặc định của trình duyệt Google Chrome vì sự tiện lợi, chỉ cần mở trình duyệt là đăng nhập được ngay. Tuy nhiên, cơ chế này tiềm ẩn rủi ro rất lớn khi máy tính bị nhiễm mã độc đánh cắp thông tin (Infostealer).

Về mặt kỹ thuật, khi người dùng bấm lưu mật khẩu trên Chrome, toàn bộ tên đăng nhập, mật khẩu và đường dẫn trang web sẽ được ghi vào một tệp cơ sở dữ liệu dạng SQLite có tên là `Login Data` nằm sẵn trong thư mục dữ liệu người dùng (`User Data`) trên ổ cứng máy tính. Để bảo vệ tệp này, Chrome sử dụng cơ chế mã hóa mặc định gắn liền với tài khoản đăng nhập của hệ điều hành (trên Windows là cơ chế DPAPI - Data Protection API) mà không yêu cầu đặt một mật khẩu khóa riêng (Master Password). Quy tắc của cơ chế này là: bất kỳ chương trình nào đang chạy dưới quyền của chính người dùng đang đăng nhập Windows thì đều được hệ điều hành cho phép giải mã dữ liệu đó một cách tự động.

Mã độc Infostealer là loại phần mềm độc hại chuyên lén lút thu thập dữ liệu nhạy cảm trên máy nạn nhân. Lập trình viên rất dễ vô tình tải phải loại mã độc này thông qua các gói thư viện mã nguồn mở bị cài cắm mã độc, tệp đính kèm giả mạo hoặc phần mềm không rõ nguồn gốc. Ngay khi chạy trên máy tính của lập trình viên, mã độc Infostealer cũng mang quyền hạn ngang với chính tài khoản Windows mà lập trình viên đang sử dụng. Lúc này, mã độc chỉ cần thực hiện hai bước đơn giản trong vài giây:

1. Truy cập thẳng vào đường dẫn cố định chứa tệp `Login Data` của Chrome trên ổ cứng.
2. Gọi trực tiếp hàm giải mã có sẵn của hệ điều hành Windows để chuyển toàn bộ mật khẩu đã mã hóa về dạng văn bản thuần (chữ đọc được bình thường), sau đó gửi toàn bộ danh sách này về máy chủ của kẻ tấn công.

Đối với lập trình viên tại PixelForge Studio, thói quen này đặc biệt nguy hiểm vì trên một trình duyệt thường lưu cùng lúc cả tài khoản cá nhân (diễn đàn, mạng xã hội) lẫn các tài khoản sống còn của dự án như GitHub Organization, AWS GameLift hay cổng thanh toán Stripe. Chỉ cần một máy tính cá nhân bị nhiễm Infostealer, kẻ tấn công sẽ nắm được toàn bộ mật khẩu của studio mà không tốn công dò đoán.

### Câu 2: Bản chất kỹ thuật của Kiến trúc Zero-Knowledge và cơ chế Client-side Decryption trên Bitwarden

Để khắc phục điểm yếu của trình duyệt, studio cần chuyển sang dùng trình quản lý mật khẩu chuyên dụng như Bitwarden với nền tảng là Kiến trúc Không-Biết-Gì (Zero-Knowledge Architecture) và cơ chế Giải mã tại máy khách (Client-side Decryption).

**1. Bản chất của Kiến trúc Zero-Knowledge (Không-Biết-Gì):**

Đây là mô hình thiết kế hệ thống mà trong đó đơn vị cung cấp dịch vụ (Bitwarden) hoàn toàn không biết bất kỳ thông tin gì về nội dung dữ liệu bên trong két sắt của người dùng, cũng như không hề biết mật khẩu chính (Master Passphrase) dùng để mở két sắt đó. Máy chủ của Bitwarden chỉ đóng vai trò như một nhà kho giữ hộ những chiếc hộp đã bị khóa chặt từ trước, còn chìa khóa mở hộp chỉ duy nhất người dùng nắm giữ.

**2. Cơ chế Mã hóa và Giải mã tại máy khách (Client-side Encryption & Decryption):**

Máy khách (Client) ở đây chính là ứng dụng Bitwarden cài trên máy tính hoặc tiện ích mở rộng trên trình duyệt của nhân viên studio. Toàn bộ quá trình đóng khóa và mở khóa dữ liệu diễn ra như sau:

- Khi người dùng nhập Master Passphrase vào ứng dụng trên máy tính, ứng dụng sẽ dùng các thuật toán tạo khóa bảo mật cao (như PBKDF2-SHA256 hoặc Argon2id) để biến mật khẩu đó thành một chìa khóa mã hóa ngay trong bộ nhớ tạm (RAM) của máy tính. Bản thân Master Passphrase gốc và chìa khóa này tuyệt đối không bao giờ được gửi nguyên bản lên máy chủ Bitwarden.
- Trước khi lưu bất kỳ mật khẩu nào lên đám mây, ứng dụng tại máy tính sẽ dùng chuẩn mã hóa đối xứng cấp độ quân sự AES-256 bit để biến toàn bộ tài khoản, mật khẩu thành một chuỗi ký tự vô nghĩa (gọi là bản mã - Ciphertext), sau đó mới gửi chuỗi đã mã hóa này lên máy chủ Bitwarden để đồng bộ.
- Khi nhân viên cần lấy mật khẩu để đăng nhập làm việc, máy chủ Bitwarden chỉ gửi trả về chuỗi dữ liệu đang bị khóa (Ciphertext). Ứng dụng Bitwarden trên máy tính của nhân viên mới dùng chìa khóa đang có trên máy để giải mã ngược lại thành mật khẩu gốc (Client-side Decryption).

**3. Giải đáp tình huống: Nếu máy chủ đám mây của Bitwarden bị hacker tấn công, dữ liệu Két sắt của PixelForge Studio có bị giải mã không? Vì sao?**

Câu trả lời là: Dữ liệu Két sắt của PixelForge Studio sẽ **KHÔNG** bị giải mã.

Lý do là vì:

- **Thứ nhất:** Trên máy chủ đám mây của Bitwarden hoàn toàn không lưu mật khẩu dạng văn bản thuần và cũng không lưu chìa khóa giải mã. Thứ duy nhất hacker lấy cắp được từ máy chủ chỉ là các khối dữ liệu đã bị khóa chặt bằng thuật toán AES-256 bit.
- **Thứ hai:** Chuẩn mã hóa đối xứng AES-256 bit sử dụng độ dài khóa 256 bit (tương đương $2^{256}$ khả năng). Với năng lực tính toán của các siêu máy tính hiện nay, việc tự giải mã trực tiếp khối dữ liệu AES-256 mà không có chìa khóa là điều bất khả thi.
- **Thứ ba:** Cách duy nhất để hacker mở được khối dữ liệu đó là phải thử đoán mò Master Passphrase của studio (tấn công vét cạn). Tuy nhiên, vì dữ liệu còn được bảo vệ bởi thuật toán kéo giãn khóa làm chậm tốc độ dò mật khẩu, nên nếu PixelForge Studio đặt một Master Passphrase đủ dài và ngẫu nhiên theo đúng chuẩn (tối thiểu 20 ký tự), hacker sẽ không thể dò ra được chìa khóa giải mã.

## PHẦN 2: THIẾT KẾ CHÍNH SÁCH & KỊCH BẢN CHUYỂN ĐỔI

### Câu 3: Phương pháp tạo Master Passphrase đạt chuẩn và ví dụ thực tế

Mật khẩu truyền thống thường được ghép từ một từ quen thuộc kèm năm sinh hoặc ký tự đặc biệt dễ đoán (như `PixelForge@2026!`). Kiểu mật khẩu này vừa ngắn vừa theo quy luật quen thuộc nên các công cụ tấn công vét cạn (Brute-Force Attack - dùng máy tính thử tự động hàng tỷ mật khẩu mỗi giây dựa trên từ điển và quy tắc thay thế chữ cái) có thể bẻ khóa rất nhanh. Ngược lại, nếu cố tạo một chuỗi ký tự ngẫu nhiên kiểu `x9#Lm2$pQ...` dài trên 20 ký tự thì con người lại rất khó nhớ, dẫn đến việc phải ghi chú hớ hênh ra giấy dán màn hình.

Để vừa dễ nhớ với con người, vừa bất khả thi với máy tính tấn công vét cạn, phương pháp tạo cụm mật khẩu (Master Passphrase) gồm 4 quy tắc sau:

1. **Ghép từ 5 đến 6 từ độc lập, không liên quan đến công ty:** Chọn các từ tiếng Việt không dấu hoặc tiếng Anh hoàn toàn ngẫu nhiên (tương tự phương pháp gieo xúc xắc chọn từ Diceware), tuyệt đối không chứa tên công ty (PixelForge), tên dự án game, tên nhân sự hay năm hiện tại.
2. **Liên kết bằng hình ảnh tưởng tượng:** Ghép các từ ngẫu nhiên đó thành một hình ảnh hoặc câu chuyện kỳ lạ, hài hước để não bộ con người ghi nhớ ngay lập tức mà không cần học vẹt từng chữ cái.
3. **Đảm bảo độ dài tối thiểu từ 20 ký tự trở lên:** Khi ghép 5 - 6 từ lại với nhau, độ dài tự nhiên của chuỗi sẽ vượt mức 20 đến 25 ký tự. Trong toán học mật mã, độ dài của mật khẩu là yếu tố quan trọng nhất làm tăng số lượng tổ hợp theo cấp số nhân, khiến máy tính vét cạn bị quá tải thời gian.
4. **Thêm dấu phân cách đặc biệt, chữ hoa và số ngẫu nhiên:** Viết hoa chữ cái đầu mỗi từ, dùng các ký tự đặc biệt khác nhau để nối giữa các từ và chèn thêm một cặp số ngẫu nhiên.

**Ví dụ cụ thể về 01 Master Passphrase đạt chuẩn:**

- **Câu chuyện tưởng tượng để nhớ:** "Con khủng long màu tím đang ngồi uống trà đá ở trạm vũ trụ số 79".
- **Master Passphrase tương ứng:** `KhungLong#Tim_UongTraDa@VuTru79!`
- **Kiểm tra độ an toàn:** Cụm mật khẩu này dài 32 ký tự (vượt yêu cầu tối thiểu 20 ký tự), bao gồm đầy đủ chữ hoa, chữ thường, chữ số và các ký tự đặc biệt (`#`, `_`, `@`, `!`). Nhân viên nắm giữ chỉ cần nhớ câu chuyện hình ảnh là gõ lại chính xác 100%, trong khi các công cụ Brute-Force phải đối mặt với không gian tổ hợp dài 32 ký tự hoàn toàn không có trong từ điển thông thường, khiến việc dò tìm bằng sức mạnh máy tính là bất khả thi.

### Câu 4: Bảng cấu hình chuẩn cho công cụ Generator trong Bitwarden

Khác với Master Passphrase mà con người phải tự nhớ để mở két sắt, mật khẩu của hơn 40 tài khoản dịch vụ (Steamworks, Epic Games, AWS, GitHub, Firebase, Stripe...) sẽ do Bitwarden tự động lưu trữ và tự động điền. Do đó, nhân viên không cần phải nhớ các mật khẩu này. Studio sẽ bắt buộc dùng công cụ sinh mật khẩu ngẫu nhiên (Generator) của Bitwarden để tạo cho mỗi tài khoản một mật khẩu riêng biệt, hoàn toàn ngẫu nhiên theo bảng cấu hình chuẩn dưới đây:

| Thông số cấu hình trên Bitwarden Generator | Giá trị thiết lập chuẩn | Giải thích mục đích áp dụng cho 40 tài khoản |
| :--- | :--- | :--- |
| Loại mật khẩu (Password Type) | Password (Ký tự ngẫu nhiên) | Tối ưu độ ngẫu nhiên cho các tài khoản dịch vụ do máy tự lưu và tự điền. |
| Độ dài mật khẩu (Length) | 24 ký tự (Trong khoảng 16 - 32 ký tự) | Đạt mức an toàn cao cho các cổng quan trọng như AWS, Stripe, GitHub; nếu dịch vụ nào giới hạn độ dài thì chỉnh tối thiểu 16 ký tự, khuyến khích tăng lên 32 ký tự cho tài khoản Root/Admin. |
| Chữ cái in hoa (A-Z) | Bật (Bắt buộc) | Tăng độ phức tạp của không gian ký tự. |
| Chữ cái in thường (a-z) | Bật (Bắt buộc) | Kết hợp cùng chữ in hoa để chống dò đoán. |
| Chữ số (0-9) | Bật (Số lượng tối thiểu: 3 số) | Đảm bảo trong chuỗi luôn có ít nhất 3 chữ số ngẫu nhiên. |
| Ký tự đặc biệt (!@#$%^&*) | Bật (Số lượng tối thiểu: 3 ký tự) | Đáp ứng yêu cầu bảo mật khắt khe của các cổng thanh toán và máy chủ đám mây. |
| Tránh các ký tự dễ nhầm lẫn (Avoid Ambiguous Characters) | Bật (Loại bỏ `I`, `l`, `1`, `O`, `0`) | Tránh nhìn nhầm giữa chữ O hoa và số 0, hoặc chữ l thường và số 1 trong trường hợp hiếm hoi kỹ thuật viên phải nhìn màn hình để nhập tay lên máy chủ console. |
| Nguyên tắc sử dụng | Mỗi tài khoản bấm "Regenerate" 1 lần mới | Đảm bảo 100% trong 40 tài khoản không có bất kỳ 2 tài khoản nào trùng mật khẩu, triệt tiêu hoàn toàn nguy cơ tấn công Credential Stuffing (dùng mật khẩu lộ ở nơi này đăng nhập nơi khác). |

### Câu 5: Vì sao tính năng Tự động Điền (Auto-fill) của Bitwarden chống tấn công Phishing giả mạo tên miền hiệu quả hơn con người?

Tấn công giả mạo (Phishing) thường đánh vào thị giác và sự mất cảnh giác của con người. Kẻ tấn công sẽ làm nhái giao diện trang đăng nhập của GitHub hoặc Steamworks giống hệt trang thật từ màu sắc, phông chữ đến logo, nhưng đặt trên một tên miền giả mạo gần giống như `githúb.com` (dùng ký tự có dấu hoặc ký tự đồng hình quốc tế - IDN Homograph Attack) hoặc đổi phần đuôi tên miền thành `steampowered.xyz`.

Nếu lập trình viên có thói quen tự gõ mật khẩu hoặc mở két sắt ra bấm "Copy - Paste" thủ công, mắt người rất dễ bỏ sót dấu sắc nhỏ trên chữ `ú` hoặc không để ý đuôi `.xyz` lúc đang làm việc căng thẳng, dẫn đến việc tự tay dán mật khẩu thật vào trang web của kẻ lừa đảo.

Ngược lại, tính năng Tự động Điền (Auto-fill) của Bitwarden bảo vệ người dùng hiệu quả hơn hẳn nhờ cơ chế kiểm tra tên miền (URI Matching) bằng máy tính:

1. **Không bị đánh lừa bởi giao diện hình ảnh:** Bitwarden hoàn toàn không quan tâm trang web có logo hay màu sắc giống GitHub hay không. Nó chỉ đọc trực tiếp chuỗi địa chỉ tên miền thực tế (Domain/URI) mà trình duyệt đang truy cập.
2. **Phân biệt chính xác từng ký tự tên miền và mã Punycode:** Khi lưu tài khoản trong Bitwarden, tên miền gốc được ghi nhận chuẩn là `github.com` hoặc `steampowered.com`. Nếu nhân viên bấm nhầm vào đường link lừa đảo `githúb.com`, ở tầng kỹ thuật của trình duyệt, tên miền có dấu này thực chất được biên dịch thành chuỗi mã Punycode là `xn--githb-rva.com`. Bitwarden đối chiếu chuỗi `xn--githb-rva.com` với `github.com`, thấy hai chuỗi hoàn toàn khác nhau nên lập tức từ chối tự động điền mật khẩu.
3. **Chặn đứng các tên miền khác đuôi:** Tương tự với `steampowered.xyz`, phần tên miền gốc (Base domain) không khớp với `steampowered.com` hay `steamgames.com` trong két sắt, nên biểu tượng Bitwarden trên trình duyệt sẽ không hiển thị tài khoản gợi ý và phím tắt tự động điền cũng không hoạt động.
4. **Cảnh báo gián tiếp cho nhân viên:** Khi nhân viên vào trang đăng nhập mà thấy Bitwarden không tự điền mật khẩu như mọi ngày, họ sẽ sinh nghi và kiểm tra lại thanh địa chỉ, nhờ đó phát hiện ra trang web giả mạo trước khi thông tin bị lộ.

### Câu 6: Quy trình lưu trữ Khóa Cấp cứu (Emergency Kit) để dự phòng sự cố nhân sự

Trong kiến trúc Zero-Knowledge, vì nhà cung cấp dịch vụ không biết mật khẩu của khách hàng nên nếu Trưởng phòng Kỹ thuật (người giữ Master Passphrase quản trị cao nhất của studio) gặp sự cố đột xuất như tai nạn, ốm nặng hoặc quên mật khẩu, Bitwarden sẽ không thể cấp lại mật khẩu qua email như các trang web thông thường. Để đảm bảo Trưởng nhóm (Team Lead) có thể khôi phục quyền truy cập cho công ty mà vẫn giữ an toàn tuyệt đối trong ngày thường, quy trình quản lý Khóa Cấp cứu (Emergency Kit) được thiết lập gồm 4 bước sau:

**Bước 1: Khởi tạo bộ hồ sơ Khóa Cấp cứu (Emergency Kit) trên bản giấy**

Ngay sau khi thiết lập tài khoản quản trị Bitwarden cho PixelForge Studio, Trưởng phòng Kỹ thuật tiến hành lập bản giấy Emergency Kit (in mẫu trắng rồi tự viết tay các thông tin mật vào, tuyệt đối không lưu thành tệp Word/PDF trên máy tính và không chụp ảnh lưu vào điện thoại). Nội dung tờ giấy bao gồm đủ 4 thông tin cốt lõi:

- Địa chỉ máy chủ đăng nhập (ví dụ: `vault.bitwarden.com`).
- Địa chỉ Email quản trị chính của tài khoản két sắt.
- Cụm mật khẩu chính (Master Passphrase) viết rõ từng ký tự.
- Mã khôi phục xác thực hai lớp (2FA Recovery Code) để mở khóa trong trường hợp không có điện thoại của Trưởng phòng Kỹ thuật để lấy mã OTP.

**Bước 2: Niêm phong bảo mật và lưu trữ vật lý theo nguyên tắc "Phân tách quyền hạn"**

- Trưởng phòng Kỹ thuật cho tờ Emergency Kit vào một phong bì dày không nhìn xuyên thấu được, dán kín mép.
- Trưởng phòng Kỹ thuật và Giám đốc Studio (CEO) cùng ký tên giáp lai đè lên mép dán của phong bì và ghi rõ ngày tháng niêm phong.
- Lập thành 02 bản niêm phong giống nhau để phòng ngừa rủi ro hỏa hoạn hoặc hư hỏng: Bản thứ nhất cất vào két sắt vật lý chống cháy tại văn phòng công ty (chỉ Giám đốc giữ chìa khóa két sắt vật lý); Bản thứ hai gửi tại két bảo hiểm ngân hàng hoặc nơi lưu trữ an toàn riêng của nhà sáng lập. Bằng cách này, ngày thường một mình Trưởng nhóm không thể tự ý lấy đọc, mà Giám đốc nếu mở ra cũng sẽ để lại dấu vết rách chữ ký niêm phong.

**Bước 3: Cấu hình song song tính năng "Quyền truy cập khẩn cấp" (Emergency Access) trên phần mềm**

Ngoài bản giấy vật lý, Trưởng phòng Kỹ thuật vào phần cài đặt của Bitwarden, sử dụng tính năng Emergency Access để thêm tài khoản Bitwarden cá nhân của Trưởng nhóm (Team Lead) làm người liên hệ khẩn cấp (Trusted Emergency Contact) với quyền Tiếp quản (Takeover) và đặt thời gian chờ (Wait Time) là 48 giờ (hoặc 72 giờ).

- Trong điều kiện bình thường, nếu Trưởng nhóm tự ý bấm yêu cầu truy cập, hệ thống sẽ gửi thông báo ngay cho Trưởng phòng Kỹ thuật để bấm từ chối.
- Nếu Trưởng phòng Kỹ thuật thực sự gặp sự cố bất khả kháng không thể phản hồi sau 48 giờ, hệ thống Bitwarden sẽ tự động cho phép Trưởng nhóm đặt lại mật khẩu và tiếp quản két sắt của studio.

**Bước 4: Quy trình mở khóa khẩn cấp và tái thiết lập an ninh sau sự cố**

Khi Trưởng phòng Kỹ thuật gặp sự cố đột xuất và công ty cần truy cập gấp vào hệ thống:

- Trưởng nhóm báo cáo Giám đốc Studio để cùng mở két sắt vật lý, kiểm tra tính nguyên vẹn của chữ ký niêm phong trên phong bì, lập biên bản mở phong bì Emergency Kit có sự chứng kiến của hai bên (hoặc kích hoạt yêu cầu Emergency Access trên phần mềm nếu không ở văn phòng).
- Trưởng nhóm dùng Master Passphrase và mã 2FA Recovery Code trong phong bì để đăng nhập và khôi phục toàn bộ quyền quản trị 40 tài khoản của studio.
- Ngay sau khi xử lý xong sự cố và bàn giao quyền quản trị mới, người tiếp quản bắt buộc phải đổi sang một Master Passphrase mới, tạo lại mã 2FA Recovery Code mới, hủy tờ Emergency Kit cũ đã bóc và lập lại một phong bì niêm phong mới theo đúng Bước 1 và Bước 2.
