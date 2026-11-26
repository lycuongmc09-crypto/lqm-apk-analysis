# libysmteam.so — PHÂN TÍCH KỸ THUẬT SÂU (RE)

> File: `lib/arm64-v8a/libysmteam.so` — 4.851.642 bytes
> ELF64 AArch64 · stripped · PIC · NX · canary · full RELRO · chỉ export **`JNI_OnLoad` @ 0x33f3fc**
> Công cụ: radare2 6.2.6 + script Python tự viết (quét adrp/add, giải mã string, dump Lua)

---

## 1. TỔNG QUAN

`libysmteam.so` là **toàn bộ engine của mod**, không chỉ là menu:

| Thành phần | Vai trò |
|---|---|
| JNI bridge | Đăng ký 6 native method cho `com.android.support.Menu`, nạp DEX nhúng trong RAM |
| Metadata resolver | Đọc `global-metadata.dat` của IL2CPP để ánh xạ **385 method game** |
| Hook engine | 22 hàm cài hook, dùng KittyMemory + `process_vm_readv` để patch code |
| Battle logic | Hook theo tướng/vật thể: Dirak, Volkath, Guardian (trừng trị), cooldown, camp |
| Lua runtime | Nhúng Lua, chạy script "soi tố", "đăng xuất nhanh", "xem hồ sơ" |
| License | Xác thực key qua `ysmteam.com`, bind thiết bị, chống VPN, kiểm tra chữ ký server |

Không có bất kỳ symbol nội bộ nào — toàn bộ tên hàm/tham số bị **XOR-obfuscate**.

---

## 2. CẤU TRÚC ELF

```
.rela.dyn      0x29d8    size 0x43cb0   (11.570 entry, 10.722 R_AARCH64_RELATIVE)
.rodata        0x47b10 → 0x1b7020
   ├─ DEX #1   0x16aca0  2.172 B      (chỉ R.java)
   └─ DEX #2   0x16b520  223.260 B    (21 class menu)
.gcc_except_table 0x1b7020
.text          0x215900 → 0x46fbb0
.plt           0x46fbb0 → 0x470980
.data.rel.ro   0x470980 (paddr) / 0x474980 (vaddr) — lệch +0x4000 từ đây
.init_array    0x4934f0   17 con trỏ
.bss           0x4a8380   size 0x1e1bc9
```

- **223 import**, không có import JNI → mọi lời gọi JNI đi qua bảng hàm `JNIEnv`.
- Compiler: clang 20.0.0 (r547379) + object cũ clang 14.0.7 / 8.0.7 / 6.0.2 (OpenSSL, curl, Lua).
- `__cxa_guard_acquire` = `0x454bd4`, `__cxa_guard_release` = `0x454d2c` — dùng cho lazy-init string.

---

## 3. `JNI_OnLoad` — luồng khởi động

```
JNI_OnLoad(JavaVM *vm, void *reserved)            @ 0x33f3fc
│
├─ vm->GetEnv(&env, JNI_VERSION_1_6)
├─ if (env == NULL || lỗi) return -1
├─ if (sub_340590() != 0) return JNI_VERSION_1_6   ; đã init rồi
├─ sub_32e678()                                    ; stub 8 byte
├─ sub_33becc(env)                                 ; MAIN INIT (3.072 B)
│     ├─ giải mã tên class + FindClass
│     ├─ RegisterNatives(com/android/support/Menu, table, 6)
│     └─ set byte[0x6843b8] = 1 nếu thành công
├─ sub_33cd2c(env) / sub_33ebc4 / sub_33d610 / sub_33d064
├─ *(void**)0x4aa000+0xd50 = vm
├─ sub_33da0c(env)
└─ if (byte[0x6843b8] == 1) sub_33a878(env)        ; MENU AUTOSTART (3.360 B)
   return JNI_VERSION_1_6
```

### 3.1 Đăng ký native — bảng dựng trên stack

```asm
x1 = 0x684540                     ; buffer .bss
x2 = &flag
x0 = sub_3389c8(env, 0x684540, &flag)   ; giải mã tên + FindClass
if (x0) {
    if (flag) byte[0x6843b8] = 1;
    env->RegisterNatives(x0, methods_on_stack, 6);   ; [env+0x6b8]
    env->DeleteLocalRef(x0);                          ; [env+0xb8]
    log("com/android/support/Menu", "RegisterNatives");
} else log("com/android/support/Menu", "FindClass");
```

➡️ **6 method** được đăng ký, nhưng `JNINativeMethod[6]` được **xây trên stack** → không thể tìm thấy bảng tĩnh trong file. Đây là lý do các scanner thông thường không thấy `GetFeatureList`, `Changes`, `nativeDangNhap`…

Chuỗi log dùng để đăng ký là plaintext: `com/android/support/Menu` @0x6d17e, `RegisterNatives` @0x7bb4b, `FindClass` @0x6763c.

---

## 4. OBFUSCATE CHUỖI

Mọi literal quan trọng (URL, tên class, tên hàm, thông báo lỗi) đều **XOR tại chỗ** khi dùng lần đầu:

```asm
adrp x9, 0x54000
ldr  d1, [x9, #0xd00]        ; key 8 byte từ .rodata 0x54d00
ldr  d0, [x8]                ; ciphertext
ldrb w10, [x8, #8]
strb wzr, [x8, #9]           ; NUL-terminate
eor  v0.8b, v0.8b, v1.8b     ; XOR 8 byte
mov  w9, 0xcd
eor  w9, w10, w9             ; byte thứ 9 XOR 0xcd
strb w9, [x8, #8]
str  d0, [x8]
```

Bảo vệ bằng `__cxa_guard_acquire/release`; destructor đăng ký qua `__cxa_atexit(dtor=0x33f528, obj=0x6843c8, dso=0x474980)`.

Các hằng ciphertext xuất hiện dày đặc trong `sub_33becc`:
`12998, 30630, 43494, 45105, 32172, 21737, 25769, 46458, 3237, 8662, 81, 53, 25, 35, 229, 19, 125, 157, 79, 145, 23, 27, 209, 41, 91, 165, 61, 11, 203, 219, 13, 73, 65, 181, 167, 233, 213, 67, 105, 87, 215, 101, 121, 237, 33, 147, 39, 189, 177, 123`.

→ Giải thích vì sao quét string chỉ thấy URL Telegram/`ysmteam.com` (nằm trong DEX) mà **không thấy endpoint API, mật khẩu, hay tên hàm ẩn**.

---

## 5. METADATA IL2CPP — CƠ CHẾ CỐT LÕI

### 5.1 Tìm `global-metadata.dat`

Hàm quanh **0x2b5900** thực hiện:

```
fopen("/proc/self/maps")           ; đọc bản đồ bộ nhớ
strstr(..., "global-metadata.dat") ; tìm vùng đã map
fopen("/proc/self/cmdline")        ; lấy package name
snprintf(path, 512, "/storage/emulated/0/Android/data/%s/files/il2cpp/Metadata/global-metadata.dat", pkg)
snprintf(path, 512, "/sdcard/Android/data/%s/files/il2cpp/Metadata/global-metadata.dat", pkg)
snprintf(path, 512, "/data/data/%s/files/il2cpp/Metadata/global-metadata.dat", pkg)
snprintf(path, 512, "/data/data/%s/files/global-metadata.dat", pkg)
snprintf(path, 512, "/data/user/0/%s/files/il2cpp/Metadata/global-metadata.dat", pkg)   ← xref 0x2b5ce4
access(path, R_OK)
```

Nếu tìm thấy vùng `global-metadata.dat` đã được game map sẵn trong RAM, mod **đọc trực tiếp từ bộ nhớ** thay vì file — tránh việc phải có quyền storage.

### 5.2 Bảng 385 method

Hàm **`0x2b9064`** (và biến thể `0x2b910c`):

```c
// w0 = index, x8 = con trỏ struct kết quả (sret)
if (index > 0x180) return;          // 384 = giới hạn
entry = 0x4aaf90 + index * 0x30;    // bảng 385 entry, stride 0x30
if (entry->resolved) return;
pthread_mutex_lock(0x4af7f4);
table = *(void**)0x4af828;          // bảng phụ, stride 0x18
if (!table[index].present) ...
```

- **Bảng chính**: `0x4aaf90`, **385 entry × 0x30 byte**, cờ `resolved` tại offset `+0x24`.
- **Bảng phụ**: con trỏ tại `0x4af828`, entry **0x18 byte**, cờ `present` tại `+0x00`.

Đây là **danh sách toàn bộ method game mà mod quan tâm**. Mỗi hook truyền vào một index nhỏ:

| Hàm hook | Index sử dụng |
|---|---|
| `0x2c5ed8` | 184, 268, 269, 273, 187, 274, 185, 186, 112, 111, 265 |
| `0x2d4eac` | 376, 378, 377, 247 |

Thông báo lỗi tương ứng (plaintext):

> `thu vien khai %lld ham, metadata khai %lld. Da tu choi %d muc de khong moc nham dia chi. Hay cap nhat global-metadata.dat.`

→ Mod **đối chiếu số lượng method giữa `libil2cpp.so` và metadata**; nếu lệch (game update) thì **từ chối một số mục** để tránh patch sai địa chỉ. Đây là cơ chế an toàn khi game đổi version.

---

## 6. HOOK ENGINE

### 6.1 Chuỗi cài hook

```
sub_21d3ec → sub_2b9230              ; khởi tạo engine
log(..., 5, "…")                     ; tag LIEMBP?
sub_3239fc  sub_2e9810  sub_2df630   ; 22 hàm cài hook
sub_2e8e0c  sub_2dda44  sub_322370
sub_2ef58c  sub_2e9d64  sub_2dcf70
sub_2ccc84  sub_2f3430  sub_2cff90
sub_2de4a4  sub_2c5ed8  sub_32c3ac
sub_3263bc  sub_2f8e34  sub_30fed4
sub_32ca4c  sub_308c44  sub_2d4098
sub_2d4eac
→ log "Cai hook xong"
```

**22 nhóm hook**, mỗi nhóm phụ trách một cụm tính năng. Bên trong mỗi hàm chỉ có các lời gọi `sub_2b9064(index)` / `sub_2b910c(index)` — **không có string**, nên không thể suy ra tên tính năng từ code.

### 6.2 KittyMemory

Chuỗi `KittyMemory` xuất hiện tại 8 điểm trong vùng `0x340b00–0x341250` (hàm khởi tạo `0x3408f4` nằm trong `.init_array`). Đây là thư viện patch bộ nhớ nổi tiếng (Android), cung cấp:

- `KittyMemoryMgr` — quản lý vùng nhớ
- `MemoryPatch` — ghi đè instruction
- `MemoryBackup` — sao lưu/khôi phục
- đọc `/proc/self/maps` để tìm base address của `libil2cpp.so`

### 6.3 Đọc/ghi bộ nhớ

String trong `.rodata`:

```
MEM_READ / MEM_WRITE
process_vm_readv
memRead err address (%p) is null
memRead err couldn't find address (%p) in any map
memRead err vung (%p, len %zu) VAT QUA bien anh xa
memWrite err ...
lay dia chi PC tru di so nay ra offset trong thu vien.
```

→ Mod tự viết lớp `memRead/memWrite` dựa trên `process_vm_readv` + parse `/proc/self/maps`, có kiểm tra biên (không đọc tràn vùng map).

---

## 7. LOGIC CHEAT — LỘ RA QUA DEBUG STRING

Mod dùng tag **`[LIEMBP]`** cho log debug (có vẻ là codename nội bộ). Các log này tiết lộ trực tiếp logic:

| Log | Ý nghĩa |
|---|---|
| `[LIEMBP] frame=%u DIRAK_VIEW objId=%llu hp=%d mana=%d` | Đọc HP/mana của đối tượng qua view Dirak |
| `[LIEMBP] frame=%u DIRAK_CURRENT objId=%llu hp=%d mana=%d` | Đọc HP/mana hiện tại |
| `[LIEMBP] frame=%u DIRAK_PROVIDER objId=%llu status=%llu` | Trạng thái provider |
| `[LIEMBP] frame=%u DIRAK_PASSIVE objId=%u configId=%u state=%u rate=%d` | Xử lý nội tại Dirak |
| `[LIEMBP] frame=%u DIRAK_VIEW_RESULT objId=%u result=%u maxHp=%d rate=%d` | Kết quả |
| `[LIEMBP] frame=%u VolkathNegativeFresh event=%llu` | Sự kiện Volkath |
| `[LIEMBP] frame=%u objId=%llu VolkathNegativeBarrier requiredAfter=%llu` | Barrier Volkath |
| `[LIEMBP] frame=%u objId=%llu configId=%u VolkathDismount529316or529317 valid=%u active=%u` | Xuống ngựa Volkath (skill 529316/529317) |
| `[LIEMBP] frame=%u GUARDIAN_SAMPLE objId=%llu flags=%02x phase=%s cdMs=%u` | Lấy mẫu Guardian (trừng trị) |
| `[LIEMBP] frame=%u GUARDIAN_STATE objId=%llu phase=%s equipValid=%u equip=%u event=%llu` | Trạng thái Guardian |
| `[LIEMBP] frame=%u GUARDIAN_RESOURCE status=%s reason=%llu detail=%llu` | Tài nguyên Guardian |
| `[LIEMBP] frame=%u cooldown sample objId=%llu current=%u duration=%u` | Hồi chiêu |
| `[LIEMBP] frame=%u cooldown direction objId=%llu direction=%llu` | Hướng hồi chiêu |
| `[LIEMBP] frame=%u CORE objId=%llu camps=%u/%u hpBps=%u d2x100=%u` | Theo dõi camp rừng |
| `[LIEMBP] frame=%u resource verified item=%llu maxActiveMs=%u cooldownMs=%u` | Xác minh item tự dùng |
| `[LIEMBP] frame=%u registry full objId=%llu` | Bảng đối tượng đầy |
| `[LIEMBP] counters publish/consume/cast/lookupFail/drop` | Thống kê hook |

Ngoài ra:

- `bbyLogic.UpdateLQ` — tên hàm game bị hook (battle logic update).
- `HEALTH_POINT`, `ATTACK_SPEED`, `COOLDOWN`, `ATTACK_SPEED` — tên thuộc tính game dùng để tra cứu.
- `attack_spell01C`, `attack_spell01D`, `attack_fly03` — animation/skill ID.

➡️ Mod **không dùng aimbot thuần tuý ở native**; nó **hook trực tiếp battle logic** để đọc HP/mana/hồi chiêu/camp và can thiệp theo tướng (Dirak, Volkath), theo vật thể (Guardian/trừng trị), theo item.

---

## 8. LUA NHÚNG

Không có bytecode `\x1bLua` — mod dùng `luaL_loadstring` với script plaintext, lưu trong `.rodata`:

| Chunk | Kích thước | Chức năng |
|---|---|---|
| `chunk_00_577dd.lua` | 3.940 B | **Soi tố cáo** — hook `SendComplantRecord`, gửi `ComplaintRecordReq`, parse `ComplaintRecordRsp` |
| `chunk_01..04_*.lua` | 1.852 B × 4 | **Đăng xuất nhanh** — máy trạng thái, dispatch `P1602/P247/P3408` |
| `chunk_05_718dc.lua` | 573 B | **Xem hồ sơ** (feature 112) — `ShowPlayerDetailInfo` |

### 8.1 Soi tố cáo

```lua
-- __ST là state machine
local nm = N.NetworkModule.instance
local t = { ComplaintRecordReq = { Uid = uid, LogicWorldID = lw } }
local p = nm:CreateDefaultCSPKG(M.CSID_CMD_COMPLAINT_RECORD_REQ, t)
nm:SendLobbyMsg(p, false, M.CSID_CMD_COMPLAINT_RECORD_RSP, __ST.nhan)
```

Khi nhận response, đọc:

```
ResultInfo.ResultId
RecordList[].BeComplaintUid
RecordList[].GameBriefInfo.GameStartTime
RecordList[].ClientReasonArray
RecordList[].DelCreditValue / DelBehaviorValue / LeftCreditValue
RecordList[].Result
RecordList[].ComplainTypeValueList[].BeComplaintCheatCount
RecordList[].PassiveEmailWarnCnt / ChgCreditType
```

Xuất ra dạng `R|uid|time|reasons|delCredit|delBehavior|result|cheatCount|leftCredit|warnCnt|chgType`.

→ **Đọc lịch sử tố cáo của người chơi khác** bằng chính API của game — hành vi xâm phạm riêng tư và lạm dụng dịch vụ.

### 8.2 Đăng xuất nhanh

4 biến thể `che=1..4` khác nhau ở điều kiện chờ:

```
che=1: chờ tới lobby rồi bắn P1602
che=2: chờ sau khi vào trận rồi bắn P247
che=3: chỉ kiểm tra trạng thái
che=4: bắn P3408 ngay khi đăng nhập
```

Đều đặt `CGCloudUpdateSystem.IsAutoLogin = false` để tránh tự đăng nhập lại.

---

## 9. XÁC THỰC / LICENSE

Phía Java (đã phân tích trước):

- `XacThuc` — UI nhập key, native `nativeDangNhap`, `nativeXoaKeyCu`, `nativeHanDung`, `nativeTrangThai`.
- `KhoKey` — AndroidKeyStore AES/GCM, alias `lgl.auth.storage.v2`, file `credential-v2.json`.
- Lỗi: `QUA_NHIEU_MAY`, phát hiện VPN, `Chữ ký máy chủ không khớp`, `Không lấy được định danh máy`.

Phía native:

- Chuỗi URL `https://ysmteam.com/getkey/YSMTEAM` nằm trong **DEX**, không phải native.
- Native dùng **OpenSSL** (X509, RSA, PKCS7) + **libcurl** để gọi server và **xác minh chữ ký phản hồi**.
- Chuỗi `secure device signature` và `-----BEGIN CERTIFICATE-----` (nhiều cert, gồm cả `vTrust ECC Root CA`) cho thấy có **certificate pinning / chain verification**.
- Bind thiết bị: tổng hợp định danh máy → gửi kèm key → server giới hạn số máy.

---

## 10. CHỐNG PHÂN TÍCH

| Kỹ thuật | Bằng chứng |
|---|---|
| Không symbol | stripped, chỉ `JNI_OnLoad` |
| Bảng native dựng runtime | `JNINativeMethod[6]` trên stack |
| XOR string + lazy init | `__cxa_guard_*`, key tại `0x54d00` |
| Chỉ số method thay vì tên | 22 hàm hook chỉ gọi `sub_2b9064(index)` |
| Kiểm tra phiên bản metadata | `thu vien khai … metadata khai …` |
| Không dùng string anti-debug | không có `frida/magisk/ptrace/xposed` — tránh bị nhận diện |
| Fallback an toàn | menu lỗi → game vẫn chạy, chỉ log |

**Điểm yếu khi reverse:**

- DEX nhúng **không mã hoá** → lấy được toàn bộ Java.
- Lua **plaintext** → lấy được toàn bộ script.
- Log debug `[LIEMBP]` **plaintext** → lộ logic từng tính năng.
- Bảng 385 method nằm ở `.bss` nhưng **index cố định**, có thể dump lúc chạy.

---

## 11. GIỚI HẠN CỦA PHÂN TÍCH TĨNH

- Chưa map được **index → tên method game** (bảng được điền lúc runtime từ metadata).
- Chưa giải mã được toàn bộ string XOR (cần emulate init hoặc chạy động).
- Chưa xác định chính xác 6 native method được đăng ký (bảng stack).
- Chưa quan sát traffic thật của `ysmteam.com` (không có Frida/adb trên máy phân tích).

---

## 12. IoC

| Loại | Giá trị |
|---|---|
| File | `lib/arm64-v8a/libysmteam.so` — 4.851.642 B |
| Export | `JNI_OnLoad` @ `0x33f3fc` |
| Tag log | `[LIEMBP]` |
| Class nhúng | `com.android.support.*` (21 class) |
| DEX offset | `0x16aca0` (2.172 B), `0x16b520` (223.260 B) |
| Bảng method | `0x4aaf90` (385 × 0x30), phụ `0x4af828` (× 0x18) |
| Hook installer | `0x21d3ec` → 22 hàm, kết thúc bằng log `Cai hook xong` |
| Hàm resolve | `0x2b9064`, `0x2b910c` |
| Metadata finder | `0x2b5900` (xref path @ `0x2b5ce4`) |
| KittyMemory | vùng `0x340b00–0x341250`, ctor `0x3408f4` |
| URL (DEX) | `https://ysmteam.com/getkey/YSMTEAM`, `t.me/ADM_YINSUMOD`, `t.me/YSMTEAM_VN` |

---

*Tài liệu phục vụ phân tích kỹ thuật và phòng chống gian lận. Không phân phối lại APK.*
