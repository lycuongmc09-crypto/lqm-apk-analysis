# MODULE MENU — PHÂN TÍCH CHI TIẾT

> Trích từ `libysmteam.so` → DEX nhúng `ysm_dex_2.dex` (223.260 B) → package `com.android.support`
> 21 class · `Menu.java` 77.928 B · `XacThuc.java` 75.566 B

---

## 1. KIẾN TRÚC TỔNG THỂ

```
libysmteam.so  (JNI_OnLoad @0x0033f3fc)
   │
   ├─ nạp DEX nhúng trong bộ nhớ (2 DEX: 2.172 B + 223.260 B)
   │     └─ com.android.support.Main.Start(context)
   │            ├─ CrashHandler.init(context)
   │            └─ GanMenu.batDau(context)
   │
   └─ GanMenu (ActivityLifecycleCallbacks)
          ├─ chờ XacThuc.daDangNhap()
          ├─ new Menu(context).ShowMenu()
          ├─ menu.ganVao(activity)        → WindowManager overlay
          └─ gắn widget phụ:
                RvCompat   (vẽ đường ngắm)
                BaoTo      (khung soi tố)
                NutTrungTri(nút trừng trị)
                NutBanDo   (nút bán đồ)
```

Toàn bộ **logic cheat nằm trong native**; Java chỉ là lớp UI + cầu nối JNI.

---

## 2. CHUỖI NẠP

### `Main.java`
```java
static { System.loadLibrary("ysmteam"); }   // nạp chính nó

public static void Start(Context context) {
    CrashHandler.init(context);
    GanMenu.batDau(context);
}
```
- Nếu `.so` không nạp được → toast "Menu: không nạp được thư viện trên máy này" và **game vẫn chạy bình thường**.
- `StartWithoutPermission()` là alias deprecated.

### `GanMenu.java`
- Đăng ký `Application.ActivityLifecycleCallbacks` vào `Application` (lấy qua `context.getApplicationContext()` hoặc reflection `ActivityThread.currentApplication()`).
- Hằng số:
  - `TRE_MO_CONG = 5000 ms` — chờ game ổn định
  - `HAN_CHOT = 20000 ms` — hạn chót mở cổng đăng nhập
  - `TRE_ON_DINH = 700 ms`
  - `TOI_DA_HONG = 3` — gắn menu thất bại 3 lần thì bỏ
- `onActivityResumed()` → `thuGan(a)` + gắn 4 widget.
- `onActivityPaused()` → gỡ widget + `menu.goRa()`.
- Nếu chưa đăng nhập → `XacThuc.moCong(activity, callback)`.

---

## 3. UI OVERLAY

`Menu` dựng bằng code, không dùng layout XML:

| Thành phần | Vai trò |
|---|---|
| `rootFrame` (FrameLayout) | gốc, nhận touch để kéo |
| `mCollapsed` (RelativeLayout) | icon thu gọn |
| `mExpanded` (LinearLayout) | bảng menu mở rộng |
| `scrollView` | chứa danh sách feature |
| `mods` | container feature (từ `GetFeatureList()`) |
| `mSettings` | container cài đặt (từ `SettingsList()`) |
| `title` / `subTitle` | native cập nhật qua `Init(context, title, subTitle)` |

Icon:
- `Icon()` → chuỗi Base64 → `BitmapFactory.decodeByteArray`.
- Hoặc `IconWebViewData()` → nạp `WebView` với `<img src="...">` (icon động).

Gắn vào màn hình:
```java
vmParams = new WindowManager.LayoutParams(
    -2, -2,
    2,              // TYPE_PHONE (overlay cũ)
    67108872,       // FLAG_NOT_FOCUSABLE | FLAG_HARDWARE_ACCELERATED
    -3);            // PixelFormat.TRANSLUCENT
vmParams.gravity = Gravity.TOP | Gravity.LEFT;
wm.addView(rootFrame, vmParams);
```
→ Cần quyền `SYSTEM_ALERT_WINDOW` (APK có xin).

Nút:
- **ẨN** → thu về icon, alpha 0.
- **THU NHỎ** → thu về icon với alpha bình thường.
- **⚙** → chuyển giữa `mods` và `mSettings`.

---

## 4. CẦU NỐI JNI

### 4.1 Native method của `Menu`
| Method | Kiểu trả về | Vai trò |
|---|---|---|
| `GetFeatureList()` | `String[]` | danh sách feature hiển thị |
| `SettingsList()` | `String[]` | danh sách mục cài đặt |
| `Icon()` | `String` | icon Base64 |
| `IconWebViewData()` | `String` | icon dạng URL/data |
| `Init(Context, TextView, TextView)` | `void` | native ghi tiêu đề/phụ đề |
| `IsGameLibLoaded()` | `boolean` | game đã nạp xong chưa |

### 4.2 Kênh giá trị duy nhất — `Preferences.Changes`
```java
public static native void Changes(
    Context ctx, int featNum, String featName,
    int intValue, long longValue, boolean boolValue, String strValue);
```
- Mọi thay đổi (toggle, seekbar, input, spinner…) đều đổ về đây.
- Native quyết định feature nào áp dụng vào game.

### 4.3 Bộ nhớ chia sẻ native → Java
| Method | Lớp | Vai trò |
|---|---|---|
| `layONho()` | `RvCompat` | `ByteBuffer` chứa dữ liệu vẽ đường ngắm |
| `layONhoBao()` | `BaoTo` | `ByteBuffer` chứa dữ liệu khung soi tố |
| `xoaToDaBiet(int)` | `BaoTo` | đánh dấu đã đọc bản ghi |

### 4.4 Nhóm `XacThuc`
`nativeDangNhap(String)`, `nativeXoaKeyCu()`, `nativeCoTuDong()`, `nativeHanDung()`, `nativeKeyDaLuu()`, `nativeLoi()`, `nativeTrangThai()`.

### 4.5 Nhóm `CrashHandler`
`nativeMarkGracefulExit()`, `nativeSetLogDir(String)`, `nativeSetMeta(String)`.

---

## 5. ĐỊNH DẠNG FEATURE LIST

Chuỗi do native trả về, mỗi phần tử:

```
[CollapseAdd_]<số>_<Loại>_<Nhãn>[_tham_số...][_True]
```

- `_True` ở cuối = **mặc định bật**.
- `CollapseAdd_` ở đầu = đưa vào nhóm gập.

| Loại | Tham số | Widget |
|---|---|---|
| `Toggle` | — | Switch |
| `SeekBar` | `min_max` | SeekBar |
| `Button` | — | Button |
| `ButtonOnOff` | — | Button bật/tắt |
| `Spinner` | `danh_sách` | Spinner |
| `InputValue` | `[max]` | hộp nhập số |
| `InputLValue` | `[max]` | hộp nhập số long |
| `InputText` | — | hộp nhập chữ |
| `CheckBox` | — | CheckBox |
| `RadioButton` | `danh_sách` | Radio |
| `Collapse` | — | nhóm gập |
| `Category` | — | tiêu đề nhóm |
| `RichTextView` | — | text HTML |
| `RichWebView` | — | WebView |
| `ButtonLink` | `url` | mở link |

Ví dụ suy ra từ danh sách chức năng:

```
101_Toggle_Map sáng
103_SeekBar_Tầm nhìn_1_100
208_SeekBar_Tầm xa ngắm (m)_5_100
217_Spinner_Chế độ AIM_...
900_Button_Đăng xuất nhanh
```

> `featureList()` trong DEX bị jadx bỏ qua ("782 instructions"); đã dump lại bằng `--show-bad-code` để xác nhận định dạng trên.

---

## 6. LUỒNG DỮ LIỆU

```
Người dùng bấm Toggle
        │
        ├─ Preferences.changeFeatureBool(name, num, bool)
        │       ├─ SharedPreferences "lgl_menu_prefs"  (lưu)
        │       └─ Preferences.Changes(...)            (gửi native)
        │
        └─ một số featNum có xử lý Java riêng:
              212 → RvCompat.batTat()
              302 → NutTrungTri.datTrungTri()
              305 → NutTrungTri.batTat()
              501 → NutBanDo.datBanDo()
              540 → NutBanDo.batTat()
              700 → BaoTo.batTat()
              705 → BaoTo.datChiTiet()
             -1  → bật/tắt lưu pref
             -2  → bật/tắt âm thanh
             -3  → mở rộng/thu gọn menu
```

Khi khởi động lại:
1. `loadPrefBool/Int/String` đọc từ `lgl_menu_prefs`.
2. Gọi `Changes(...)` **phát lại** giá trị đã lưu vào native.
3. Nếu `loadPref == false` → không phát lại (trừ featNum âm).

---

## 7. CÁC WIDGET PHỤ

### `RvCompat extends View` — vẽ đường ngắm
- Feature **212**.
- Đọc `ByteBuffer` từ `layONho()`, vẽ khung/đường bằng `Canvas`.
- Màu: chính `#00DCFF`, viền `#DCDCDC`, lỗi trắng.

### `BaoTo extends View` — khung "soi tố"
- Feature **700** (bật), **705** (chi tiết), **703/704** (cỡ/độ mờ).
- Đọc `ByteBuffer` từ `layONhoBao()`, có con trỏ `VT_SEQ` và `xoaToDaBiet(seq)`.
- Tự lưu vị trí khung.

### `NutTrungTri extends View` — nút trừng trị
- Feature **302** (bật), **304** (kiểu), **305** (nút nổi), **306/307** (cỡ/độ mờ).
- 3 màu kiểu: teal `#2DD4BF`, đỏ `#F87171`, vàng `#FBBF24`.
- Có `Handler` riêng để cập nhật/vẽ lại.

### `NutBanDo extends View` — nút bán đồ
- Feature **501** (bật), **540** (nút nổi), **541/542** (cỡ/độ mờ).
- Màu bật teal, tắt đỏ.

### `DangXuat` — feature **900**
- Dialog xác nhận, chỉ gửi lệnh xuống native khi người dùng đồng ý.

### `ListBanDo` / `BoListBanDo` — danh sách món
- Feature **530–537** (món 1–8), **543** (chế độ), **544** (chọn list), **545** (sửa/xoá).
- Lưu tên list + 8 món, có kiểm tra tên trùng/độ dài.

### `AmThanh` — âm thanh
- Tự sinh PCM (không có file âm thanh), phát bằng `AudioTrack`.
- Bật/tắt bằng feature **-2**.

### `PresetAim` — preset theo tướng
- Bảng `BANG` 12 dòng (0 = mặc định + 11 tướng).
- Kẹp giá trị seek, định dạng hiển thị (độ lệch `0.260`, màu 1–256).

### `CrashHandler`
- Bắt exception Java + native, ghi log, `nativeMarkGracefulExit()`.
- Gửi kèm danh sách thư viện đã nạp (`/proc/self/maps`).

---

## 8. LƯU TRỮ

| Loại | Nơi lưu |
|---|---|
| Cấu hình menu | `SharedPreferences` **`lgl_menu_prefs`** |
| Migration | từ `<package>_preferences`, cờ `_lgl_migrated` |
| License key | AndroidKeyStore `lgl.auth.storage.v2` + `credential-v2.json` (`lgl_auth/`) |
| Prefs license | `lgl-auth-settings-v2` (`auto_login`, `legacy_closed`) |

---

## 9. NHẬN XÉT

**Điểm đáng chú ý:**
1. Menu dùng **overlay `TYPE_PHONE`** + `SYSTEM_ALERT_WINDOW`, sống xuyên Activity.
2. Toàn bộ quyết định cheat nằm trong native → **Java layer không chứa logic hack**, chỉ UI.
3. `Preferences.Changes` là **kênh duy nhất** để Java → native; native có toàn quyền quyết định.
4. Dữ liệu vẽ (đường ngắm, khung soi tố) đi qua **shared `ByteBuffer`** do native cấp → native chủ động bơm dữ liệu.
5. Menu có **cơ chế chịu lỗi**: không nạp được .so / không gắn được overlay → game vẫn chạy, chỉ mất menu.
6. Tên class/hàm tiếng Việt không dấu (`GanMenu`, `NutBanDo`, `BaoTo`, `XacThuc`…) → nhóm phát triển người Việt.

**Giới hạn phân tích:**
- Chưa có `GetFeatureList()` thực tế (do native sinh runtime) → danh sách đầy đủ lấy từ string `muc N` trong `.so`.
- Chưa chạy động nên chưa xác nhận từng feature tác động vào hàm game nào.

---

*Tài liệu phục vụ phân tích kỹ thuật, không phân phối lại APK.*
