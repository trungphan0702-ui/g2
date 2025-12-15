# Audio Measurement Toolkit

> **Tuyên bố kiến trúc (BẮT BUỘC)**
>
> - **GUI contract bất biến:** `GUI_D_3_2_1.py` là file GUI tham chiếu. **Không đổi layout / không đổi tên widget/tab/button / không tách GUI sang file khác.**
> - **Backend không phụ thuộc Tkinter:** backend **không được import** `tkinter`, `messagebox`, Tk variables, hoặc đụng vào state của GUI.
> - **Cửa ngõ duy nhất:** GUI **chỉ được gọi** backend qua **`backend/contracts.py`**:
>   - `from backend.contracts import ...`
>   - Cấm mọi import kiểu `from analysis...`, `from audio...`, `import analysis...`, `import audio...` trong GUI.
> - Backend có thể có nhiều module nội bộ (`analysis/`, `audio/`, `utils/`), nhưng **không expose trực tiếp** cho GUI.

---

## 1) Mục tiêu dự án

Toolkit đo lường audio gồm 2 chế độ:

1) **Offline**: phân tích WAV có sẵn (ref/meas) để tính THD / Compressor / Attack-Release, Compare (align/latency/delta metrics).
2) **Loopback (Realtime)**: phát stimulus → đi qua thiết bị/chain ngoài → thu về → phân tích theo **chunk** để GUI vẽ realtime và xuất artifact.

Các nhóm phép đo (áp dụng cho cả Offline & Loopback):

- **THD**
- **Compressor characteristics**
- **Attack/Release**
- **Compare (align & latency, delta metrics)**
- **Audio I/O (list/validate/playrec/wav read/write)**

---

## 2) Cấu trúc thư mục (chuẩn)

Gợi ý cấu trúc (có thể đã tồn tại một phần trong repo):

```
project/
  GUI_D_3_2_1.py                # GUI contract (bất biến layout)
  backend/
    __init__.py                 # chỉ export contracts (khuyến nghị)
    contracts.py                # API public duy nhất cho GUI
  analysis/                     # DSP (private)
  audio/                        # device + wav + play/rec (private)
  utils/                        # helpers internal (private)
  tests/
```

**Quy ước public/private:**
- **Public**: chỉ `backend/contracts.py` (và `backend/__init__.py` nếu dùng).
- **Private/internal**: mọi thứ còn lại (nếu cần, prefix `_` để tránh gọi nhầm).

---

## 3) Backend API Contract (luật chữ ký)

### 3.1 Hai mẫu entrypoint bắt buộc

**Mẫu 1 – Sync (nhanh)**
```py
def run_xxx(request: XxxRequest) -> XxxResult:
    ...
```

**Mẫu 2 – Async/Worker (dài, realtime)**
```py
def start_xxx(
    request: XxxRequest,
    *,
    stop_event: threading.Event | None = None,
    on_progress: Callable[[ProgressEvent], None] | None = None,
    on_log: Callable[[LogEvent], None] | None = None,
) -> XxxHandle:
    ...
```

### 3.2 Stop/Cancel bắt buộc
- Task dài **phải hỗ trợ** `stop_event` và/hoặc `handle.cancel()`.
- Khi bị dừng: raise `CancelledError`.

### 3.3 Progress/log callback
- `on_log(LogEvent)` dùng để stream log.
- `on_progress(ProgressEvent)` dùng để stream tiến độ và **stream dữ liệu realtime**.

---

## 4) Chuẩn hoá output (để GUI dễ hiển thị)

Mọi `Result` **phải có**:

- `summary: dict[str, Any]` — giá trị chính để GUI show
- `plots: list[PlotSpec]` — **chỉ mô tả data/metadata**, KHÔNG trả figure/matplotlib object
- `artifacts: list[Artifact]` — file output + metadata truy vết
- `logs: list[str]` — optional

### 4.1 PlotSpec (không vẽ trong backend)
`PlotSpec` chứa:
- `kind`, `title`
- `series` (danh sách series x/y + label)
- `meta` (fs, units, downsample, …)

GUI/plot-window có thể render PlotSpec theo cách riêng.

---

## 5) Chuẩn hoá Artifact + Metadata (BẮT BUỘC)

Mọi phép đo (THD / Compressor / Attack-Release, cả Offline và Loopback) **phải có khả năng** xuất:
- CSV (ít nhất metrics/summary)
- và/hoặc WAV (tx/rx/processed) tùy feature

### 5.1 Artifact schema đề xuất (nhất quán)
Trong `Artifact.meta` **tối thiểu** nên có:

- `run_id`: str (UUID hoặc timestamp+rand)
- `timestamp`: ISO-8601 string
- `feature`: `"thd" | "compressor" | "attack_release" | "compare" | "loopback_record"`
- `mode`: `"offline" | "loopback_realtime"`
- `sample_rate`: int
- `channels`: int
- `input_device`: dict|None (index, name, hostapi nếu có)
- `output_device`: dict|None
- `stimulus`: dict (freq_hz, amp, amp_max, rms_win_ms, duration, sweep params… tùy phép đo)
- `config`: dict (hmax, max_lag_seconds, windowing, FFT params, … nếu có)
- `source_files`: dict (ref_wav_path, tgt_wav_path, input_wav_path, …)
- `notes`: str|None

> Mục tiêu: **không nhầm** giữa các lần chạy / chế độ / thiết bị.

---

## 6) Streaming realtime theo chunk (BẮT BUỘC cho Loopback)

Khi chạy loopback realtime, backend phải stream dữ liệu theo chunk để GUI vẽ realtime:

- Mỗi lần gọi `on_progress(...)` trong phase streaming:
  - **BẮT BUỘC** `meta={"chunk": <index>}`
  - payload phải là **dữ liệu thuần** (vd: spectrum, envelope, gain reduction…)
  - KHÔNG trả figure object

Ví dụ:
```py
on_progress(ProgressEvent(
    phase="streaming",
    percent=None,
    message="spectrum",
    meta={"chunk": i, "fs": fs, "freq_axis_hz": freq_axis, "mag_db": mag_db},
))
```

---

## 7) Danh sách entrypoints (contract “chuẩn”)

> (Tên hàm cụ thể sẽ đúng theo `backend/contracts.py`)

### 7.1 Audio I/O
- `run_list_devices(ListDevicesRequest) -> ListDevicesResult`
- `run_validate_device(ValidateDeviceRequest) -> ValidateDeviceResult`
- `run_wav_read(ReadWavRequest) -> ReadWavResult`
- `run_wav_write(WriteWavRequest) -> WriteWavResult`
- `start_loopback_record(LoopbackRecordRequest, ...) -> LoopbackRecordHandle`

### 7.2 Compare
- `run_compare(CompareRequest) -> CompareResult`

### 7.3 THD
- Offline: `run_thd_offline(ThdOfflineRequest) -> ThdResult`
- Loopback realtime: `start_thd_loopback(ThdLoopbackRequest, ...) -> ThdHandle`

### 7.4 Compressor
- Offline: `run_compressor_offline(CompressorOfflineRequest) -> CompressorResult`
- Loopback realtime: `start_compressor_loopback(CompressorLoopbackRequest, ...) -> CompressorHandle`

### 7.5 Attack/Release
- Offline: `run_attack_release_offline(AROfflineRequest) -> ARResult`
- Loopback realtime: `start_attack_release_loopback(ARLoopbackRequest, ...) -> ARHandle`

---

## 8) Mapping GUI → Contract (nguyên lý chạy)

### 8.1 Nguyên tắc
- GUI chỉ làm:
  1) đọc input từ widget
  2) tạo `Request` dataclass
  3) gọi `run_*` hoặc `start_*`
  4) hiển thị `Result.summary`, list `Result.artifacts`, render `Result.plots` nếu có

### 8.2 Quy tắc thread
- Với `start_*`: GUI chạy trong thread/worker để không block Tk loop.
- UI update phải dùng `master.after(0, ...)`.
- Khi bấm Stop: `stop_event.set()` hoặc `handle.cancel()`.

---

## 9) Quy tắc refactor & “dọn hàm dư thừa”

- Bất kỳ hàm DSP/IO rải rác, trùng chức năng, chữ ký không thống nhất:
  - chuyển vào internal (prefix `_` hoặc module `_internal/`)
  - hoặc gộp về **1 entrypoint duy nhất** trong `contracts.py`
- GUI **không được nhìn thấy** các hàm đó.

---

## 10) Quickstart cho dev

### 10.1 Chạy GUI (dev)
```bash
python GUI_D_3_2_1.py
```

### 10.2 Nguyên tắc khi thêm tính năng mới
1) Implement DSP/IO trong `analysis/` hoặc `audio/` (private).
2) Thêm/điều chỉnh 1 entrypoint (hoặc mở rộng request/result) trong `backend/contracts.py`.
3) GUI chỉ nối callback của button → gọi entrypoint tương ứng (không đổi layout).

---

## 11) Anti-patterns (cấm)

- Import `analysis/*` hoặc `audio/*` trực tiếp trong GUI.
- Backend import `tkinter` hoặc dùng `messagebox`.
- Backend trả về `matplotlib.figure.Figure` hoặc object không-serializable.
- Không có `stop_event`/cancel cho task realtime.
- Artifact không có metadata → dễ nhầm run.

---

## 12) Tài liệu liên quan

- `backend/contracts.py`: contract chuẩn (source of truth)
- `GUI_D_3_2_1.py`: GUI immutable contract (layout bất biến)
