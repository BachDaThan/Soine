# Soine — Kế hoạch phát triển (v2, dựa trên code thật của Y-App và Adaptive-MoE)

> Soine hiện chỉ có `LICENSE` (MIT). Kế hoạch này bắt đầu từ việc đọc code hai repo có sẵn, không phải từ giả định.

## 0. Những gì code thật cho thấy (điều chỉnh so với kế hoạch cũ)

| Phát hiện | Hệ quả cho Soine |
|---|---|
| `yui.cpp` (1421 dòng) chỉ đọc định dạng riêng `.yui` (magic `YUI1`), chỉ sinh văn bản, prompt cố định `User:/Yui:`. **Không có** GGUF, vision, embedding. | "Runtime thống nhất cho mọi model" là không thể. Vision/embedding cần runtime khác (ONNX/TFLite). `yui.cpp` chỉ làm **chat/luyện hội thoại**. |
| Model Yui là MoE nhỏ tự huấn luyện (~20M–1B, tiếng Việt, dữ liệu ít). | Không đủ để "phân tích hồ sơ/ảnh" chất lượng cao. Phần phân tích phải dùng **luật + thuật toán + model chuyên dụng nhỏ**, không dùng LLM của Yui. |
| README Adaptive-MoE tự ghi: adaptive routing **không** hơn top-k cố định; thứ hiệu quả là **elastic training** (một model chạy được ở nhiều k). | Trên điện thoại: chọn `k` theo cấu hình máy (k=1 máy yếu, k=2 máy khá). Không đầu tư thêm vào adaptive routing cho Soine. |
| Y-App: ~2.5k dòng, Flutter + FFI + isolate, chat/lịch sử/model/cài đặt, CI build APK + Windows. README ghi **Dart chưa được build thử**. | Bước 0 bắt buộc: làm cho nó build và chạy trên máy thật trước khi thêm gì. |
| Y-App lưu chat bằng `shared_preferences`. | Không đủ cho hồ sơ, ảnh, quyền riêng tư. Phải chuyển sang SQLite. |
| Trình tải model: có `.part` nhưng **không** Range/resume, **không** SHA-256, chỉ kiểm magic `.yui`, token truyền thẳng. | Nâng cấp thành Model Manager thật (mục 3). |
| `yui.cpp` đang nằm ở **hai** repo (Y-App/native và Adaptive-MoE). | Chọn một nguồn duy nhất (Adaptive-MoE), Soine kéo về lúc build. Tránh hai bản lệch nhau. |

## 1. Nguyên tắc

1. **App dùng được khi chỉ có một người dùng** (chế độ khách, không tài khoản).
2. **Internet là kênh chính**. Ngoại tuyến là bổ sung, chỉ trên Android.
3. **Dữ liệu cá nhân ở lại máy** cho đến khi người dùng chủ động chia sẻ từng trường.
4. **Mỗi giai đoạn ra một APK kiểm thử được** qua GitHub Actions (đúng quy trình bạn đang dùng với Kính/Yui).
5. **Đo thật trên điện thoại thật** trước khi tuyên bố tính năng chạy được.

## 2. "AI" trong Soine thực sự là gì (phân vai trung thực)

| Chức năng | Cách làm | Cần LLM? |
|---|---|---|
| Chất lượng ảnh (nét, sáng, tương phản, đóng khung) | Thuật toán cổ điển: phương sai Laplacian (độ nét), histogram (phơi sáng), phát hiện khuôn mặt on-device (ML Kit hoặc TFLite) để kiểm tra mặt có trong khung/đủ lớn | Không |
| Gợi ý chọn ảnh hồ sơ | Xếp hạng theo điểm chất lượng ở trên, hiển thị lý do | Không |
| Profile Insight | Bộ câu hỏi có cấu trúc → thẻ (tag) → mẫu câu gợi ý; LLM nhỏ chỉ để diễn đạt lại sau | Tùy chọn, sau |
| Matching | Lọc cứng → điểm tương đồng trên thẻ có cấu trúc (mục 5) | Không |
| Luyện hội thoại | Kịch bản định sẵn trước; sau đó dùng `yui.cpp` với model SFT riêng cho tình huống này | Có (giai đoạn 4) |

Không suy luận tính cách, cảm xúc hay "độ hợp nhau" từ khuôn mặt. Không dùng sinh trắc học để so khớp.

## 3. Model Manager (nâng cấp từ `ModelStore` của Y-App)

Giữ: nhập file, tải link, `.part`, xóa, mmap. Thêm:

- **Manifest** `models.json` đặt trong repo HF của bạn (`BachDaThan/Model`): `id, name, task, runtime, format, repo, revision, file, sha256, size, license, min_ram_mb, arch`. Trường chưa biết hiển thị "chưa xác minh".
- **Resume** bằng header `Range` khi tải (cần thêm; hiện chưa có).
- **SHA-256** (gói `crypto`) khi manifest có giá trị; không khớp → xóa `.part`.
- **Kiểm tương thích trước khi tải**: dung lượng trống, RAM ước tính (`size × hệ số`), ABI (arm64), `runtime` có trong app không.
- **Không dựa vào đuôi file**: đọc header (`YUI1`, và sau này `ONNX`/`TFL3`) rồi chọn runtime.
- **Token HF** lưu bằng `flutter_secure_storage`, không ghi log.
- **Trạng thái** model: chưa tải / đang tải / tạm dừng / đang xác minh / sẵn sàng / đang dùng / lỗi / có bản mới.
- Không xóa model đang nạp; không tự tải khi mở app.

Runtime theo tác vụ:

| Tác vụ | Runtime |
|---|---|
| Chat, luyện hội thoại | `yui.cpp` (`.yui`, v2 q4 cho điện thoại) |
| Phát hiện mặt | ML Kit on-device hoặc TFLite |
| Embedding hồ sơ (nếu cần) | ONNX/TFLite (ví dụ e5-small đa ngôn ngữ), **không** thuộc MVP |

## 4. Lộ trình (mỗi mốc = một bản APK kiểm thử được)

### Mốc 0 — Fork nền và làm cho build chạy (ưu tiên số 1)
- Copy `lib/`, `native/`, workflow, installer từ Y-App vào Soine; đổi tên gói (`com.bachdathan.soine`), `kRepo`, nhãn app.
- Kéo `yui.cpp` từ Adaptive-MoE lúc build (script trong workflow) thay vì giữ bản copy.
- Tạo keystore riêng cho Soine + 4 secret như README Y-App.
- **Xong khi:** Actions xanh, cài APK lên máy thật, chat với một model `.yui` thành công.

### Mốc 1 — Lớp dữ liệu và Model Manager
- Chuyển lưu trữ sang SQLite (`drift` hoặc `sqflite`): hồ sơ, ảnh (đường dẫn), lịch sử phân tích, cài đặt, quyền chia sẻ.
- Chế độ khách; màn hình Quyền riêng tư (xem/xóa từng loại dữ liệu).
- Model Manager đầy đủ như mục 3.
- **Xong khi:** đóng/mở app, cập nhật đè bản cũ, dữ liệu và model còn nguyên; model hỏng/không tương thích bị từ chối rõ ràng.

### Mốc 2 — Local AI cá nhân (không cần tài khoản)
- Chấm điểm ảnh + gợi ý chọn ảnh (mục 2).
- Bộ câu hỏi hồ sơ → thẻ → bản xem trước hồ sơ, sửa/xóa được.
- Hồ sơ mẫu gắn nhãn **"dữ liệu giả lập"**, không bao giờ trộn với người thật.
- **Xong khi:** người mới mở app, phân tích 1 ảnh, tạo hồ sơ, xem kết quả, không cần mạng.

### Mốc 3 — Matching qua Internet
- Backend: dùng nền tảng có Postgres + Auth + Row-Level Security (ví dụ Supabase free tier). HF Spaces free không hợp để làm DB chính (đĩa tạm thời, Space ngủ khi không dùng).
- Chính sách quyền riêng tư thực thi **ở phía server** (RLS), không chỉ ở client.
- Luồng: đăng ký → chọn trường công khai → gợi ý + lý do → "quan tâm" hai chiều → nhắn tin → chặn/báo cáo/thu hồi.
- Giới hạn tần suất, phân trang, chống spam.
- **Xong khi:** 2 tài khoản thử đi hết luồng; tài khoản bị chặn không liên hệ được; người dùng xóa tài khoản thì dữ liệu server bị xóa.

### Mốc 4 — Luyện hội thoại và nhóm thử nghiệm nhỏ
- Kịch bản định sẵn (mở lời, giữ nhịp, kết thúc lịch sự), nhãn "nhân vật AI".
- Sau đó: SFT một model Yui cho tình huống này (dùng `train_sft.py`/`sft_format.py` có sẵn), xuất `.yui` q4.
- Thử với 20–50 người tự nguyện; đo tỷ lệ hoàn thành hồ sơ, kết nối hai chiều, quay lại sau 7 ngày.

### Mốc 5 — Khám phá ngoại tuyến (chỉ Android)
- Dùng **Google Nearby Connections** (BLE + Wi-Fi do hệ thống chọn) thay vì tự viết BLE/Wi-Fi Direct: ít lỗi hơn, cùng quyền `BLUETOOTH_SCAN/ADVERTISE/CONNECT`, `NEARBY_WIFI_DEVICES`.
- Quảng bá chỉ một **ID tạm đổi liên tục**; hồ sơ chỉ trao đổi **sau khi hai bên xác nhận** trên kênh đã xác thực/mã hóa. Gói quảng bá BLE rất nhỏ nên không chứa hồ sơ/embedding.
- Tắt mặc định, có hạn dùng, có nút thu hồi.
- **Xong khi:** 2 máy thật tìm thấy nhau không cần Internet; đo pin/giờ và tỷ lệ kết nối.

### Ngoài lộ trình — Mesh nhiều chặng
Chỉ làm nếu mốc 5 có số liệu tốt và có đủ thiết bị thử. Nếu làm: TTL, chống lặp, chống phát lại, ký từng gói. Hiện coi là nghiên cứu, không cam kết.

## 5. Matching (cải tiến so với công thức tuyến tính cũ)

1. **Lọc cứng trước**: tuổi ≥ 18 và trong khoảng hai bên chọn, mục tiêu kết nối tương thích, khoảng cách **thô** (vùng/quận, không lưu tọa độ chính xác), chưa chặn nhau.
2. **Chấm điểm** trên các nhóm người dùng tự khai: sở thích, giá trị/mục tiêu, phong cách giao tiếp, lối sống. Bắt đầu bằng độ tương đồng Jaccard/cosine trên thẻ — đơn giản, giải thích được.
3. **Giải thích**: mỗi gợi ý liệt kê thẻ chung ("cùng thích chơi game co-op, cùng ưu tiên trò chuyện chậm").
4. Trọng số cấu hình được, hiển thị điểm như **gợi ý**, không phải kết luận khoa học. Cung hoàng đạo: tùy chọn giải trí, tắt mặc định, không vào điểm.
5. Đánh giá bằng hành vi thật (tỷ lệ "quan tâm" hai chiều, hội thoại kéo dài) ở mốc 4, rồi mới chỉnh trọng số hoặc thêm embedding.

## 6. Quyền riêng tư, an toàn, pháp lý

| Mục | Mặc định |
|---|---|
| Ảnh gốc, hồ sơ | Chỉ trên máy cho đến khi người dùng chọn chia sẻ |
| Vị trí | Tắt; chỉ vùng thô khi cần |
| BLE/Nearby, relay | Tắt |
| Lịch sử AI | Bật/tắt và xóa được |
| Token, khóa | `flutter_secure_storage` |

- **Giới hạn 18+**: tự khai tuổi khi đăng ký, chặn người dưới 18; không ghép/nhắn tin với trẻ vị thành niên. Có báo cáo + chặn từ đầu (cũng là yêu cầu của kho ứng dụng đối với app có nội dung người dùng).
- Ảnh người dùng tải lên server cần kiểm duyệt (ít nhất báo cáo + gỡ; về sau thêm lọc tự động).
- Dữ liệu cá nhân người dùng Việt Nam: xem Nghị định 13/2023/NĐ-CP về bảo vệ dữ liệu cá nhân (nên tự kiểm tra lại văn bản hiện hành trước khi phát hành); cần có đồng ý rõ ràng, quyền xóa dữ liệu, chính sách quyền riêng tư công khai.
- Thông báo rõ nhân vật hội thoại là AI; không giả vờ có người ở gần.

## 7. Cấu trúc repo đề xuất

```
Soine/
  lib/
    core/          # db, secure storage, settings, update check
    features/
      models/      # Model Manager (manifest, tải, xác minh)
      photo/       # chấm điểm ảnh
      profile/     # bộ câu hỏi, thẻ, xem trước
      matching/    # client matching + giải thích
      chat/        # luyện hội thoại (FFI yui)
      offline/     # Nearby (mốc 5)
  native/          # CMake/NDK; yui.cpp kéo từ Adaptive-MoE lúc build
  .github/workflows/build-app.yml
  docs/PLAN.md
```

## 8. Chưa làm

Mesh ngay từ đầu · LLM lớn chạy nền · đồng bộ toàn bộ dữ liệu cá nhân lên server · định vị chính xác · embedding/vision model nặng khi chưa chứng minh cần · hứa AI "đoán" tính cách hay tình yêu · iOS (Y-App hiện chỉ Android + Windows).

## 9. Nghiệm thu (rút gọn)

- [ ] Mốc 0: Actions xanh, APK chạy trên máy thật, chat `.yui` được
- [ ] Mốc 1: cập nhật đè giữ dữ liệu; model hỏng bị từ chối; tải tiếp được sau mất mạng; SHA-256 sai thì xóa
- [ ] Mốc 2: toàn bộ luồng dùng được khi tắt mạng và không có tài khoản; xóa ảnh/hồ sơ/lịch sử hoạt động
- [ ] Mốc 3: kết nối hai chiều; chặn/báo cáo; xóa tài khoản xóa dữ liệu server; RLS chặn truy cập trái phép (có test)
- [ ] Mốc 4: có số liệu hoàn thành hồ sơ / kết nối hai chiều / quay lại 7 ngày
- [ ] Mốc 5: 2 máy thật tìm nhau ngoại tuyến; không lộ hồ sơ qua quảng bá; đo pin và tỷ lệ kết nối

## 10. Việc làm ngay (theo thứ tự)

1. Mốc 0 (fork, build xanh). Mình có thể tạo sẵn bộ file để bạn đẩy lên Soine.
2. Quyết định backend cho mốc 3 (Supabase hay tự dựng).
3. Chốt bộ câu hỏi hồ sơ và danh sách thẻ (tiếng Việt) — đây là "dữ liệu" quyết định chất lượng matching.
