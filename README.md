# BÁO CÁO PHÂN TÍCH APK — LIÊN QUÂN MOBILE 1.64.1.7

> Mẫu: `Liên Quân Mobile_1.64.1.7.apk` — 228.178.602 bytes
> SHA-256: `69319E9C62E1E1A183A437244A8EABD780A9069908D5CDC8F077AEF9C11ECA09`
> Ngày phân tích: 26/11/2026 · Phương pháp: static only (jadx 1.5.1, apktool 2.11.1, radare2 6.2.6, REA MCP, GitHub Actions)

---

## 1. KẾT LUẬN NHANH

Đây **KHÔNG phải** APK phát hành chính thức của Garena. Đây là bản **repack** của Liên Quân Mobile 1.64.1.7 (Garena VN), được **ký lại bằng chứng chỉ `Android Debug`** và **chèn một mod menu cheat trả phí** của nhóm **YSM Team** (người Việt).

| Hạng mục | Kết quả |
|---|---|
| Loại | Game APK repack + native mod menu |
| Chữ ký | `CN=Android Debug` tự ký (không phải Garena/Tencent) |
| Mod menu | `libysmteam.so` (4,63 MB) + DEX nhúng 223 KB |
| Bản chất menu | Cheat có **license key**, bind thiết bị, server `ysmteam.com` |
| Mức độ tin cậy | **UNTRUSTED** — lớp native bị mã hoá/đóng gói, không kiểm chứng được toàn bộ |
| Nguy cơ chính | Ban tài khoản, lộ thông tin thiết bị, bề mặt IPC hở, không có cập nhật an toàn |

**Khuyến nghị:** không đăng nhập tài khoản chính, không nạp tiền, không cấp quyền thừa. Nếu chỉ muốn "xem có gì" thì báo cáo này đã đủ; nếu định chạy thì nên dùng máy/account phụ.

---

## 2. ĐỊNH DANH GÓI

| Trường | Giá trị |
|---|---|
| Package | `com.garena.game.kgvn` |
| versionName | `1.64.1.7` |
| versionCode | `176857764` |
| compileSdk | 36 |
| Application class | `com.garena.game.kgtw.NGameApplication` |
| Launcher | `com.garena.game.kgtw.SGameActivity` (exported) |
| Engine | Unity **2022.3.5f1**, **IL2CPP**, Release, StripEngineCode=1 |
| Fingerprint | `META-INF/com.android.games.engine.build_fingerprint` = `2022.3.5f1;IL2CPP;Release;StripEngineCode:1;OptimizedFramePacing:0;` |
| AGP | 8.6.0 |
| ABI | chỉ `arm64-v8a` |
| Kênh | `AOVInstallChannel=google`, `assets/channel.ini` → `CHANNEL=73213123` |

---

## 3. CẤU TRÚC APK

- **2.079 entry**, giải nén ~237 MB.
- **5 DEX**, tổng **27.393 class**:
  - `classes.dex` 8,6 MB — 8.336 class
  - `classes2.dex` 9,4 MB — 8.000 class
  - `classes3.dex` 9,2 MB — 10.199 class
  - `classes4.dex` 1,1 MB — 857 class
  - `classes5.dex` 720 B — 1 class
- **36 thư viện native** arm64. Điểm đáng chú ý:
  - `libResources.so` (55,2 MB) — **không phải ELF**, là **archive 7z** chứa `libil2cpp.so`, `libunity.so`, `global-metadata.dat` và 21 lib phụ.
  - `assets/game_resources.png` (112,2 MB) — **cũng là archive 7z**, chứa 3.458 file dữ liệu game (`Databin/Client/*`, `Ages/`, `Blueprint/`, `Config/`...).
  - `libysmteam.so` (4,63 MB) — **mod menu**.
  - `libanort.so`, `libextractor.so`, `libpandora.so`, `libwukong.so`, `libgem.so` — loader/giải nén/anti-cheat.
  - `libanogs.so`, `libaovai.so` — anti-cheat Tencent ACE (nằm trong payload 7z).
- `assets/res.data` chứa **danh sách MD5** của các lib mong đợi, ví dụ:
  - `libil2cpp.so` → `bfaca44d543d5be3adb674cd9b1e3ec9`
  - `libunity.so` → `1cdd63cf9a72d63331c0cf0235a89ff8`
  - `global-metadata.dat` → `86d7804c34a21cf30ceb45b7b0488298`

> Việc `libil2cpp.so`/`libunity.so` **không tồn tại trực tiếp** trong APK, mà nằm trong một file `.so` đóng gói 7z, là dấu hiệu **packer/loader** đi kèm mod.

---

## 4. CHỮ KÝ SỐ — BẰNG CHỨNG REPACK

REA `inspect_apk_signature`:

| Trường | Giá trị |
|---|---|
| Scheme | v1 + v2 + v3 (không có v3.1) |
| Nhất quán | Cả 3 scheme khớp nhau, `self_consistent=true` |
| Chứng chỉ | `CN=Android Debug, OU=Android, O=US, L=US, ST=US, C=US` |
| Loại | Self-signed, RSA |
| Serial | `056C1A15` |
| Hiệu lực | 2016-10-23 → 2044-03-10 |
| Cert SHA-256 | `1e08a903aef9c3a721510b64ec764d01d3d094eb954161b62544ea8f187b5953` |
| File v1 | `META-INF/ANDROIDD.SF`, `META-INF/ANDROIDD.RSA` |

**Kết luận:** APK đã bị **giải nén, sửa, đóng gói lại và ký bằng debug key**. Không thể coi là bản phát hành chính thức, và cũng **không có bảo đảm toàn vẹn cập nhật** (bất kỳ ai giữ key đều có thể phát hành bản mới).

---

## 5. LỚP LOADER `com.ano.gshell`

Đây là lớp "shell" được chèn vào, chỉ có **4 class**:

```
com/ano/gshell/AnoApplication
com/ano/gshell/GP6Service
com/ano/gshell/GP7Service
com/ano/gshell/GP7Worker
```

### 5.1 `AnoApplication`
- `extends Application`.
- Static block:
  ```java
  System.loadLibrary("anort");
  System.loadLibrary("ysmteam");
  ```
- Native methods: `gp6ioctl(String)`, `gp7ioctl(String)`, `handleLoad(...)`, `handleLoadV22(...)`, `ioctl(int, String)`, `initialize(packageName, filesDir, nativeLibraryDir, sourceDir, splitSourceDirs)`.
- `attachBaseContext()` gọi `initialize(...)`; nếu trả về khác 0 → ném `Exception("initialize failed.")`.

### 5.2 Ba service **exported, không permission**

| Service | process | exported | permission |
|---|---|---|---|
| `com.ano.gshell.GP6Service` | `:GP6Service` | **true** | **none** |
| `com.ano.gshell.GP7Service` | `:GP7Service` | **true** | **none** |
| `com.ano.gshell.GP7Worker` | `:GP7Worker` | **true** | **none** |

- Mỗi service có `intent-filter` với action **trùng tên class**.
- `onStartCommand()` đọc `intent.getStringExtra("ioctl")` rồi gọi thẳng `AnoApplication.gp6ioctl()` / `gp7ioctl()`.

➡️ **Bất kỳ app nào trên máy cũng có thể `startService(new Intent("com.ano.gshell.GP7Service").putExtra("ioctl", "..."))`** để đẩy lệnh vào loader. Đây là **bề mặt tấn công IPC thật**, không phải chỉ lý thuyết.

### 5.3 Đường khởi động menu
- Application class trong manifest là `NGameApplication`, kế thừa `MultiDexApplication` (đã xác minh bằng parser DEX: class_def 7200, super_type 4341).
- `AnoApplication` **không** phải app class và **không có subclass** nào kế thừa nó.
- Tuy nhiên jadx hiển thị comment override `// com.ano.gshell.AnoApplication` trên `NGameApplication.onCreate/attachBaseContext` → DEX có dấu hiệu **đã bị patch tay** (invoke-super trỏ sang `AnoApplication` trong khi class_def vẫn ghi `MultiDexApplication`).
- Đường kích hoạt chính xác **chưa chốt 100%**: có thể qua patch invoke-super, hoặc qua service exported ở process riêng.

### 5.4 Activity chain
- `SGameActivity extends com.ngame.resources.ResourceDecompressActivity` — exported, launcher, decompress resource + check MD5, lỗi gửi CrashSight.
- `SGameRealActivity extends com.unity3d.player.UnityPlayerActivity` — load plugin qua reflection: TDataMaster, gcloud, MSDKCore, MSDKUnityAdapter, CrashSight, nettool.

---

## 6. MOD MENU — `libysmteam.so`

### 6.1 Đặc điểm kỹ thuật
- ELF64 AArch64, stripped, PIC, NX, canary, full RELRO.
- **Chỉ export một symbol: `JNI_OnLoad` @ `0x0033f3fc`** — mẫu native menu injection kinh điển.
- Bundle kèm OpenSSL + libcurl + Lua; biên dịch bằng clang 20.0.0 (2025) lẫn object clang 14/8/6.
- Nhúng **2 DEX**:
  - offset `0x16aca0`, 2.172 B — chỉ có `R.java`
  - offset `0x16b520`, **223.260 B** — **toàn bộ menu**, 21 class `com.android.support.*`
- Chuỗi nhận diện: `ysmteam`, `com/android/support/Main`, `com/android/support/Menu`.

### 6.2 Liên hệ / hạ tầng
- `https://t.me/ADM_YINSUMOD`
- `https://t.me/YSMTEAM_VN`
- `https://ysmteam.com/getkey/YSMTEAM` — **server cấp license key**

### 6.3 Cơ chế bản quyền (`XacThuc` + `KhoKey`)
- UI đăng nhập key + native `nativeDangNhap(String)`, `nativeXoaKeyCu()`.
- **Bind thiết bị**: lỗi `QUA_NHIEU_MAY` ("quá số máy").
- **Phát hiện VPN**: "Đang bật VPN — hãy tắt VPN rồi thử lại".
- Kiểm tra key hết hạn / bị khoá / sai game / server bảo trì.
- Kiểm tra **chữ ký server** ("Chữ ký máy chủ không khớp") và **định danh máy** ("Không lấy được định danh máy").
- Lưu key bằng **AndroidKeyStore AES/GCM**:
  - alias `lgl.auth.storage.v2`
  - AAD `lgl.auth.storage:v2`
  - file `credential-v2.json` trong `getNoBackupFilesDir()/lgl_auth`
  - prefs `lgl-auth-settings-v2` (`auto_login`, `legacy_closed`)

### 6.4 Cơ chế chèn menu (`GanMenu`)
- Đăng ký `Application.ActivityLifecycleCallbacks`.
- Chờ `XacThuc.daDangNhap()` → `new Menu(appContext).ShowMenu()` → `menu.ganVao(activity)`.
- Hằng số: `HAN_CHOT=20000 ms`, `TRE_MO_CONG=5000`, `TRE_ON_DINH=700`, `TOI_DA_HONG=3`.
- Gắn các thành phần: `RvCompat`, `BaoTo`, `NutTrungTri`, `NutBanDo`; tag `Mod_Menu`.
- Nếu không nạp được: log `KHONG NAP DUOC libysmteam.so - menu se khong chay, nhung game van chay binh thuong.` → **game vẫn chạy bình thường khi menu tắt**.

### 6.5 Danh sách chức năng (từ string `muc N`)

| Nhóm | ID | Chức năng |
|---|---|---|
| Thông tin đối thủ | 100 | Hiện độ thân mật |
| | 102 | Hiện chiêu cuối địch |
| | 106 | Hiện hồi chiêu |
| | 107 / 108 | Bậc rank / số sao |
| | 109 | Hiện rank tướng địch |
| | 112 | Xem hồ sơ khi cấm chọn |
| Bản đồ / tầm nhìn | 101 | **Map sáng** |
| | 103 | **Tầm nhìn** |
| Mở khoá | 104 | **Mở khoá rank cao** |
| | 400 | **Mở khoá trang phục** |
| Aimbot | 201 | Bật ngắm kỹ năng (AIM) |
| | 202/203/204 | Ngắm chiêu 1/2/3 |
| | 207 | Chỉnh sẵn theo tướng |
| | 208 / 209 | Tầm xa ngắm (m) / độ lệch |
| | 211 / 212 / 213 | Kiểu AIM / vẽ đường ngắm / khi nào vẽ |
| | 215 / 216 / 217 | Màu tia (xanh lá/xanh dương) / chế độ AIM |
| Trợ chiến | 300 / 302 | Tự dùng bộc phá / trừng trị |
| | 304 / 305 / 306 | Kiểu trừng trị / nút nổi / cỡ nút |
| | 308 | Hayate C3 tự dùng phụ trợ (Ngất ngư/Suy nhược/Thanh tẩy) |
| | 600 | **Tự né chiêu địch** |
| Kinh tế | 501 | Tự bán & mua lại |
| | 520 | Đổi nút bấm |
| | 530–537 | Món 1–8 |
| | 541–545 | Cỡ nút / độ mờ / chế độ / chọn list / sửa-xoá |
| Soi tố cáo | 700 | **Soi tố tự động** |
| | 702 | Soi mấy trận gần nhất |
| | 703 / 704 / 705 | Cỡ khung / độ mờ / hiện chi tiết |
| Khác | 900 | Đăng xuất nhanh |

### 6.6 Preset AIM theo tướng

| Tướng | Chiêu | Tầm (m) | Lệch |
|---|---|---|---|
| Elsu | 2 | 25 | 0.260 |
| Gildur | 2 | 15 | 0.555 |
| Grakk | 2 | 13 | 0.695 |
| Slimz | 1 | 19 | 1.392 |
| Yue | 2 | 14 | 0.664 |
| Natalya | 2 | 13 | 1.256 |
| Enzo | 2 | 11 | 0.532 |
| Stuart | 1 | 17 | 0.818 |
| Florentino | 1 | 9 | 0.277 |
| Volkath | 2 | 11 | 0.532 |
| Raz | 2 | 12 | 0.639 |

### 6.7 Lua nhúng — "soi tố"
Menu hook hàm `AOV.Complaint.ComplaintSys.SendComplantRecord`, dựng request:

```lua
{ ComplaintRecordReq = { Uid = uid, LogicWorldID = lw } }
```

gửi bằng `M.CSID_CMD_COMPLAINT_RECORD_REQ` qua `N.NetworkModule.instance:SendLobbyMsg`, rồi parse `ComplaintRecordRsp`:

- `BeComplaintUid`, `ClientReasonArray`
- `DelCreditValue`, `DelBehaviorValue`, `LeftCreditValue`
- `ComplainTypeValueList`, `BeComplaintCheatCount`
- `PassiveEmailWarnCnt`, `ChgCreditType`

→ **Lạm dụng API tố cáo của game để xem lịch sử bị tố của người khác** (vấn đề riêng tư, ngoài ý muốn của nhà phát hành).

Ngoài ra còn Lua tự đăng xuất: đặt `CS.Assets.Scripts.GameSystem.CGCloudUpdateSystem.IsAutoLogin=false`, bắn UI event `P1602`/`P247`.

---

## 7. ĐÓNG GÓI / MÃ HOÁ PAYLOAD (chưa giải được)

- `libResources.so` và `game_resources.png` là **7z archive với `Method = 22`** — codec **không chuẩn**.
- 7-Zip 26.04 và p7zip **liệt kê được** tên/kích thước nhưng **giải nén thất bại** `Unsupported Method`.
- `libextractor.so` export:
  - `ExtractFile`, `ExtractFile2`, `ExtractInit`, `ExtractUnInit`
  - `stub_decrypt_elf`
  - `OodleLZ_Decompress`
  - `ELF_HOOK_InitializeMethodMetadata`, `ELF_HOOK_InitializeAllMethodMetadata`
- Dữ liệu tại offset 32: `00 00 00 04 00 00 00 00 54 33 c5 00 8c 0c 01 0d ...`, entropy cao (256/256 byte trong 64 KB đầu).

➡️ Nhiều khả năng payload được **nén bằng OodleLZ + mã hoá stub ELF**. Chưa có codec/key nên **không thể xác minh MD5** trong `res.data` và **không thể dump `global-metadata.dat`/`libil2cpp.so`**.

---

## 8. ĐÁNH GIÁ RỦI RO

| Rủi ro | Mức | Ghi chú |
|---|---|---|
| **Ban tài khoản** | Cao | Anti-cheat ACE (`libanogs`, `libaovai`), CrashSight, TP/GameSafe vẫn có mặt; hành vi aim/map/auto-dodge dễ bị phát hiện |
| **Không toàn vẹn cập nhật** | Cao | Self-signed debug key; ai giữ key cũng phát hành được bản mới |
| **IPC hở** | Trung bình–cao | 3 service exported, **không permission**, nhận lệnh `ioctl` từ app bất kỳ |
| **Payload mã hoá** | Không xác định | Không thể loại trừ stealer/backdoor trong lớp native đã đóng gói |
| **Quyền rộng** | Trung bình | `SYSTEM_ALERT_WINDOW`, storage, location, RECORD_AUDIO, GET_TASKS, WRITE_SETTINGS... |
| **Dữ liệu thiết bị** | Trung bình | License bind máy + gửi định danh lên `ysmteam.com`; có Telegram liên hệ |
| **Riêng tư người khác** | Trung bình | "Soi tố" đọc lịch sử tố cáo qua API game |
| **Vi phạm ToS** | Cao | Chắc chắn vi phạm điều khoản Garena |

**Những gì chưa thể kết luận:**
- Không thấy URL stealer/token rõ ràng trong lớp string đã đọc, nhưng **không đủ để nói "sạch"** vì payload bị mã hoá.
- Chưa chạy động (không có Frida/adb trên máy), nên chưa quan sát được traffic thực tế của `ysmteam.com`.

---

## 9. CÁCH TỰ KIỂM CHỨNG

```bash
# 1. Chữ ký
apksigner verify --print-certs "Liên Quân Mobile_1.64.1.7.apk"

# 2. Danh sách file + payload 7z
unzip -l "Liên Quân Mobile_1.64.1.7.apk" | head -50
7z l lib/arm64-v8a/libResources.so

# 3. Chuỗi nhận diện menu
strings -a lib/arm64-v8a/libysmteam.so | grep -E 'ysmteam|t\.me|muc '

# 4. Decompile lớp loader
jadx --no-res --single-class com.ano.gshell.AnoApplication app.apk

# 5. Kiểm tra export service
aapt dump xmltree app.apk AndroidManifest.xml | grep -A5 GP7Service
```

---

## 10. IoC / DẤU VẾT

- Package: `com.garena.game.kgvn` — versionCode `176857764`
- Cert SHA-256: `1e08a903aef9c3a721510b64ec764d01d3d094eb954161b62544ea8f187b5953`
- File lạ: `lib/arm64-v8a/libysmteam.so`, `lib/arm64-v8a/libanort.so`, `lib/arm64-v8a/libextractor.so`, `assets/game_resources.png` (7z), `lib/arm64-v8a/libResources.so` (7z)
- Class: `com.ano.gshell.*`, `com.android.support.Menu`, `com.android.support.XacThuc`, `com.android.support.GanMenu`
- Service: `com.ano.gshell.GP6Service`, `GP7Service`, `GP7Worker`
- URL: `https://ysmteam.com/getkey/YSMTEAM`, `https://t.me/ADM_YINSUMOD`, `https://t.me/YSMTEAM_VN`
- Native export: `JNI_OnLoad` @ `0x0033f3fc` trong `libysmteam.so`
- Native methods: `gp6ioctl`, `gp7ioctl`, `handleLoad`, `handleLoadV22`, `initialize`

---

## 11. PHỤ LỤC — NGUỒN DỮ LIỆU ĐÃ TẠO

| File | Nội dung |
|---|---|
| `AndroidManifest.decoded.xml` | Manifest giải mã (906 dòng) |
| `ysm_menu_src/` | Java của 21 class menu |
| `GP6Service.java`, `GP7Service.java`, `GP7Worker.java` | Loader services |
| `AnoApplication.java`, `NGameApplication.java` | App/loader |
| `SGameActivity.java`, `SGameRealActivity.java` | Activity chain |
| `artifact.zip` | Jadx sources các package chọn lọc + manifest + res |
| `libres_verify.txt` | Rỗng — do codec 7z chưa hỗ trợ |

---

*Báo cáo chỉ phục vụ mục đích phân tích kỹ thuật. Không phân phối lại APK.*
