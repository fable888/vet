# GHI CHÚ — Quy trình tìm và vet token cho `pairs.txt`

Ba nơi làm việc:

| Việc | Thư mục | Chạy bằng |
|---|---|---|
| Tool 1 — quét pool | `C:\Users\Admin\Documents\scan-pancake-v2-token-pool` | cmd hoặc PowerShell (Windows) |
| Tool 2 — vet token | `C:\Users\Admin\Documents\vet-bsc-token` | cmd hoặc PowerShell (Windows) |
| Bot | `~/bsc-sandwich` (WSL Ubuntu) | cửa sổ Ubuntu |

Luồng: **Tool 1** tìm token có pool V2 (WBNB/USDT) đang có swap thật → **Tool 2** kiểm
từng token (đọc source + mua/bán thử trên fork) → dòng PASS copy vào `pairs.txt` của
bot → bot tự đo lại tax mỗi 10 phút.

Mỗi tool có `.env` riêng, làm 1 lần:

```
BSC_HTTP=https://bsc.rpc.blxrbdn.com,https://rpc-bsc.48.club,https://bsc-dataseed1.bnbchain.org,https://bsc-dataseed1.defibit.io,https://bsc-rpc.publicnode.com
BSCSCAN_API_KEY=<key Etherscan V2 của bạn>
```

Không dán `.env` hay key vào chat. Nếu lỡ dán, tạo key mới ngay.

---

## PHẦN 1 — Tool 1: quét token có pool + có swap thật

### Bước 1.1 — Mở thư mục, build (chỉ lần đầu hoặc sau khi sửa code)

```
cd C:\Users\Admin\Documents\scan-pancake-v2-token-pool
cargo build --release
```

Kỳ vọng: `Finished release ...`. Lỗi thì đọc dòng đỏ đầu tiên.

### Bước 1.2 — Chạy quét 24 giờ

```
cargo run --release -- --hours 24 --existing \\wsl$\Ubuntu\home\dmin\bsc-sandwich\pairs.txt --out out\run_YYYYMMDD
```

- `--hours 24`: quét 24 giờ gần nhất (BSC 0,45 s/block → ~192.000 block). Lần đầu thử
  `--hours 2` cho nhanh.
- `--existing …pairs.txt`: để cột `in_existing` báo token nào bot đã có. Nếu đường dẫn
  `\\wsl$` không đọc được, thay bằng `\\wsl.localhost\Ubuntu\home\dmin\bsc-sandwich\pairs.txt`.
- `--out out\run_YYYYMMDD`: thư mục kết quả, đặt theo ngày để không đè.

Thời gian: ~15–20 phút. Nếu thấy nhiều dòng `429`/`-32005` là RPC bị giới hạn, tool tự
đổi URL và chạy tiếp — không cần làm gì.

Tham số hay chỉnh:

| Muốn | Thêm |
|---|---|
| Chỉ pool WBNB | `--quote wbnb` |
| Chỉ pool USDT | `--quote usdt` |
| Victim tối thiểu 0,1 BNB thay vì 0,05 | `--min-swap-bnb 0.1` |
| Pool phải ≥ 50 BNB | `--min-reserve-bnb 50` |
| Lấy 500 dòng thay vì 300 | `--top 500` |

### Bước 1.3 — Đọc kết quả

Trong `out\run_YYYYMMDD\`:

- `discover_v2.tsv` — mở bằng Excel (tab-separated). Cột quan trọng:
  - `swaps_ge_min`: số lần có người mua ≥ 0,05 BNB trong 24 h → **đây là "mồi"**. Càng cao càng tốt.
  - `reserve_quote`: thanh khoản pool (BNB hoặc USDT).
  - `verified`: `yes` mới xét tiếp.
  - `proxy`: `0` mới xét tiếp (trừ token Binance-Peg lớn).
  - `contract_name`: tên contract trên BscScan.
    - `FlapTaxTokenV2/V3` = **có tax theo thiết kế → bỏ luôn**.
    - `Token` = mẫu four.meme (thường 0 tax) → ứng viên tốt, vẫn phải vet.
  - `in_existing`: `vetted` = bot đã có; `chua_vet` = có trong pairs.txt nhưng chưa điền ngày; `khong_co` = mới.
- `pairs.candidates.txt` — cùng danh sách, đã đúng định dạng `pairs.txt`, ô `vetted` để trống.

### Bước 1.4 — Chọn danh sách đưa sang Tool 2

Trong Excel: lọc `verified = yes`, `proxy = 0`, `contract_name` không chứa `Flap`,
`swaps_ge_min ≥ 300`. Sắp giảm dần theo `swaps_ge_min`. Lấy 30–50 dòng đầu.

Cách nhanh không cần Excel (PowerShell, tạo file 50 dòng đầu):

```powershell
Get-Content out\run_YYYYMMDD\pairs.candidates.txt | Select-Object -First 50 | Set-Content out\run_YYYYMMDD\top50.txt
```

(Danh sách này chưa lọc Flap/proxy — Tool 2 sẽ tự FAIL chúng, chỉ tốn thời gian chạy.)

---

## PHẦN 2 — Tool 2: vet từng token

### Bước 2.1 — Mở thư mục, build

```
cd C:\Users\Admin\Documents\vet-bsc-token
cargo build --release
```

### Bước 2.2 — Chạy vet danh sách từ Tool 1

```
cargo run --release -- --in C:\Users\Admin\Documents\scan-pancake-v2-token-pool\out\run_YYYYMMDD\top50.txt --date 2026-09-16 --out out\meme_YYYYMMDD
```

- `--date`: ngày sẽ ghi vào ô `vetted` của dòng PASS. Dùng ngày hôm đó.
- `--out`: thư mục kết quả riêng cho lần chạy.

Mỗi token mất ~10–30 giây (đọc source + 6 bước mua/bán thử trên fork). 50 token ≈ 10–20 phút.

### Bước 2.3 — Vet lại danh sách blue-chip đang có trong pairs.txt của bot

Token Binance-Peg (ETH, BTCB, XRP, USDC…) là proxy của Binance → mặc định bị FAIL. Với
riêng nhóm này cho phép proxy:

```
cargo run --release -- --in \\wsl$\Ubuntu\home\dmin\bsc-sandwich\pairs.txt --date 2026-09-16 --allow-proxy --out out\groupA_YYYYMMDD
```

**Không** dùng `--allow-proxy` cho danh sách meme.

### Bước 2.4 — Đọc kết quả

Trong `out\<tên>\`:

| File | Nghĩa | Bạn làm gì |
|---|---|---|
| `pairs.pass.txt` | Token đạt. Ô `vetted <ngày>`, `tax 0/0`, `owner …` đã điền sẵn | Copy sang `pairs.txt` của bot |
| `pairs.review.txt` | Không rõ: source có dấu hiệu (fee/blacklist/cooldown…) nhưng mua/bán thử vẫn OK, và owner còn quyền | Đọc tay trên BscScan (mục 2.5). Nghi ngờ thì bỏ |
| `pairs.fail.txt` | Loại: tax > 0, honeypot, cooldown, proxy, chưa verify, pool mỏng… | Bỏ. Không cần đọc thêm |
| `vet_report.tsv` | Bảng đầy đủ (mở Excel) | Xem khi cần lý do chi tiết |
| `vet_steps.tsv` | Số từng bước D1–D6 | Chỉ xem khi debug |

Ý nghĩa cột `reasons` hay gặp:

- `buy_tax_bps=300` → mua mất 3 %. Loại.
- `honeypot` → mua được, bán không được. Loại.
- `cooldown` → cùng block bán bị chặn, block sau bán được. Sandwich không dùng được. Loại.
- `same_wallet_same_block` → cùng ví mua rồi bán trong 1 block bị chặn. Loại.
- `max_tx` / `anti_whale` → mua lớn (1 % reserve) bị chặn hoặc bị bớt. Loại.
- `proxy` → contract nâng cấp được. Loại (trừ Binance-Peg với `--allow-proxy`).
- `not_verified` → không có source. Loại.

### Bước 2.5 — Đọc tay nhóm REVIEW (chỉ nhóm này)

Mở `https://bscscan.com/address/<token>#code`, Ctrl+F các từ trong cột `static_groups`
của `vet_report.tsv` (ví dụ `setFee`, `blacklist`, `cooldown`, `pause`). Hỏi 3 câu:

1. Hàm đó có `onlyOwner` không? Owner còn là ví thật (`owner = active …`)?
2. Nếu owner bật hàm đó lên, bot có bị chặn bán hoặc mất tiền không?
3. Token có đáng (nhiều `swaps_ge_min`) để chấp nhận rủi ro đó không?

Trả lời "có, có, không" → bỏ. Nếu giữ: tự sửa dòng trong `pairs.review.txt`: xóa phần
`REVIEW: …` ở cuối, điền `vetted 2026-09-16` vào ô trống, rồi coi như PASS.

---

## PHẦN 3 — Đưa vào bot (WSL)

### Bước 3.1 — Gộp file

Mở cửa sổ Ubuntu:

```bash
cd ~/bsc-sandwich
cp pairs.txt state/pairs.backup_$(date +%Y%m%d).txt

W=/mnt/c/Users/Admin/Documents/vet-bsc-token/out
cat $W/groupA_YYYYMMDD/pairs.pass.txt $W/meme_YYYYMMDD/pairs.pass.txt > state/merged.txt
sed -i 's/\r$//' state/merged.txt          # bỏ ký tự CRLF của Windows
grep -c '^0x' state/merged.txt             # số dòng token
grep -c 'vetted 2026' state/merged.txt     # phải bằng số trên
```

Nếu có dòng REVIEW bạn đã tự duyệt ở 2.5, thêm vào `state/merged.txt` trước khi ghép.

### Bước 3.2 — Thay phần thân pairs.txt

Giữ nguyên phần header (các dòng `#` đầu file), thay toàn bộ dòng token bằng `merged.txt`:

```bash
grep '^#' pairs.txt > state/header.txt
cat state/header.txt state/merged.txt > pairs.txt
grep -c '^0x' pairs.txt
git add pairs.txt && git commit -m "pairs.txt: vet $(date +%Y-%m-%d) — $(grep -c '^0x' pairs.txt) token"
```

### Bước 3.3 — Kiểm bot nhận đúng

```bash
scripts/paper_run.sh --minutes 3 --port 18799
```

Nhìn 3 chỗ trong output:

- `pairs.txt: tong=… vetted=…` — `vetted` phải bằng số dòng bạn vừa thêm.
- `/api/pairs` (hoặc `curl -s 127.0.0.1:18799/api/pairs` khi bot đang chạy): cột
  `buy_bps`/`sell_bps` = 0, `honeypot` = false.
- `grep pair.vet_fail logs/bot.jsonl | tail` — dòng nào xuất hiện ở đây là token bot đo
  ra có tax thật → xóa khỏi `pairs.txt`, commit lại.

---

## PHẦN 4 — Làm lại định kỳ

- **Hàng tuần**: chạy lại Tool 1 (`--existing` trỏ pairs.txt), lấy token mới lọt top,
  vet bằng Tool 2, thêm vào. Token cũ không còn `swaps_ge_min` đáng kể thì xóa cho nhẹ.
- **Khi bot báo `pair.vet_fail`**: xóa token đó, không cần vet lại.
- **Trước khi lên live**: chạy lại Tool 2 cho toàn bộ `pairs.txt` với ngày mới, vì PASS
  chỉ đúng tại thời điểm đo.

---

## Sự cố thường gặp

| Hiện tượng | Nguyên nhân | Làm gì |
|---|---|---|
| `The system cannot find the path specified` | đường dẫn `--in`/`--existing` sai | `Test-Path <đường dẫn>` phải ra `True` |
| Tool 1 báo `429`/`-32005` liên tục | RPC free bị giới hạn | để tool tự đổi URL; hoặc giảm `--hours` |
| Tool 2 nhiều dòng `REVIEW: static_unavailable` | Etherscan API lỗi / hết quota key | chạy lại sau vài phút, hoặc `--skip-static` (chỉ dựa mua/bán thử) |
| Tool 2 `REVIEW: sim_error` | RPC lỗi giữa fork | chạy lại riêng token đó |
| Bot: `vetted=0` dù đã copy | file có CRLF hoặc thiếu ngày sau chữ `vetted` | `sed -i 's/\r$//' pairs.txt`, kiểm định dạng `vetted 2026-09-16` |
| Bot: `pair.vet_fail` cho token PASS | token đổi tax sau khi vet | xóa token, commit |
