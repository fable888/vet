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

## PHẦN 3 — Đưa vào bot (WSL) — QUY TRÌNH ĐÃ CHẠY THẬT 2026-09-16

Nguyên tắc: **thay toàn bộ phần thân `pairs.txt`** bằng các file `pairs.pass.txt` của
Tool 2, KHÔNG dùng `cat >>` (append) — append từng làm dính 2 địa chỉ vào 1 dòng khi
dòng cuối thiếu ký tự xuống dòng, và không xóa được dòng cũ đã FAIL.

Có 2 lần chạy Tool 2 cần gộp:
- `groupA…` = danh sách blue-chip trong `pairs.txt` cũ, chạy với `--allow-proxy`
  (chỉ để token Binance-Peg qua).
- `meme…` = danh sách từ Tool 1, chạy KHÔNG `--allow-proxy`.

### Bước 3.1 — Gộp 2 file PASS, bỏ proxy ngoài Binance-Peg, bỏ trùng

Mở cửa sổ Ubuntu:

```bash
cd ~/bsc-sandwich
cp pairs.txt state/pairs.backup_$(date +%Y%m%d_%H%M).txt      # backup

grep '^#' pairs.txt > state/header.txt                        # giữ phần comment đầu file

W=/mnt/c/Users/Admin/Documents/vet-bsc-token/out
cp $W/groupA*/pairs.pass.txt state/passA.txt                  # đổi tên 
cp $W/meme*/pairs.pass.txt   state/passM.txt                  # đổi tên 
sed -i 's/\r$//' state/passA.txt state/passM.txt              # bỏ CRLF của Windows

# --allow-proxy cho qua MỌI proxy; chỉ giữ proxy của Binance-Peg (admin 0xd2f9…),
# bỏ 6 proxy của bên khác:
grep -vE '# (U|QQQB|PEX|BSCPAD|USD1|SANTOS) \|' state/passA.txt > state/passA_final.txt

# gộp A + M, bỏ trùng theo địa chỉ (kể cả ",0xUSDT"), mỗi dòng 1 token:
cat state/passA_final.txt state/passM.txt | awk '$1 ~ /^0x/ && !seen[tolower($1)]++' > state/body.txt
```

### Bước 3.2 — Ghép header + thân, kiểm tra, commit

```bash
cat state/header.txt state/body.txt > pairs.txt

grep -c '^0x' pairs.txt              # số dòng token (lần này: 89)
grep -c 'vetted 2026' pairs.txt      # = số trên + 2 (2 dòng comment header cũng có chữ này)
awk '$1 ~ /^0x/ && gsub(/0x[0-9a-fA-F]{40}/,"&")>2' pairs.txt   # PHẢI RỖNG: không dòng nào dính 2 token

git add pairs.txt && git commit -m "pairs.txt: <N> token PASS (groupA + meme), thay toàn bộ thân file"
```

Nếu `ls $W/` không có thư mục tên `groupA…`/`meme…` thì sửa 2 dòng `cp` theo tên thật.

### Bước 3.3 — Bật quote USDT nếu có pool USDT

Dòng dạng `0xToken,0x55d398…7955 # …` là pool USDT. Bot chỉ nhận chúng khi
`scan_quote_usdt = true`:

```bash
sed -i 's/^scan_quote_usdt *=.*/scan_quote_usdt = true/' config.toml
grep -n 'scan_quote_usdt' config.toml
git add config.toml && git commit -m "config: bật scan_quote_usdt"
```

### Bước 3.4 — Kiểm bot nhận đúng (bot tự đo lại tax bằng revm)

```bash
scripts/paper_run.sh --minutes 5 --port 18799
grep -c '"event":"pair.vet_fail"' logs/bot.jsonl
grep '"event":"pair.vet_fail"' logs/bot.jsonl | tail -5
```

Đọc:
- Dòng `pairs.txt: tong=… vetted=…` — `vetted` phải bằng số dòng token.
- `pair.vet_fail` = 0 → toàn bộ danh sách được bot xác nhận lại. Nếu > 0: token đó thực
  ra có tax → xóa dòng khỏi `pairs.txt`, commit lại.
- `/api/skips`: `honeypot_or_tax` phải ≈ 0, `unprofitable`/`victim_would_revert` > 0
  (chứng tỏ candidate đã tới bước tính lãi/lỗ).

### Kết quả lần làm 2026-09-16 (để đối chiếu)

| Nguồn | Vào Tool 2 | PASS | REVIEW | FAIL | Giữ |
|---|---|---|---|---|---|
| groupA (`--allow-proxy`) | 93 | 82 | 6 | 5 (tax: FLOKI, Jager, GIGGLE, 雪球, LIGHT) | 76 (bỏ 6 proxy ngoài Binance) |
| meme50 (Tool 1) | 50 | 23 | 1 (CAT/USDT) | 26 (tax ẩn, chặn mua DEX, cấm cùng block, honeypot) | 13 mới (10 trùng groupA) |
| **Tổng `pairs.txt`** | | | | | **89** (7 pool USDT) |

Bài học: `contract_name = Token` KHÔNG đảm bảo sạch (Pro, IBS, VTA, BNBDXS là `Token`
mà có tax 2,5–10 %). Chỉ tin bước mua/bán thử của Tool 2.

### Log nguyên văn lần chạy 2026-09-16 (đây là "đúng" trông như thế nào)

```
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ grep '^#' pairs.txt > state/header.txt
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ cp /mnt/c/Users/Admin/Documents/vet-bsc-token/out/groupA*/pairs.pass.txt state/passA.txt
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ sed -i 's/\r$//' state/passA.txt state/passM.txt
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ grep -vE '# (U|QQQB|PEX|BSCPAD|USD1|SANTOS) \|' state/passA.txt > state/passA_final.txt
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ cat state/passA_final.txt state/passM.txt | awk '$1 ~ /^0x/ && !seen[tolower($1)]++' > state/body.txt
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ cat state/header.txt state/body.txt > pairs.txt
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ grep -c '^0x' pairs.txt; grep -c 'vetted 2026' pairs.txt
89
91
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ awk '$1 ~ /^0x/ && gsub(/0x[0-9a-fA-F]{40}/,"&")>2' pairs.txt
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ git add pairs.txt && git commit -m "pairs.txt: 89 token PASS (groupA 76 + meme 13), thay toàn bộ thân file"
[master 10f5d1c] pairs.txt: 89 token PASS (groupA 76 + meme 13), thay toàn bộ thân file
 1 file changed, 77 insertions(+), 93 deletions(-)
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ grep 'vetted 2026' pairs.txt | grep -vc '^0x'
2
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ sed -i 's/^scan_quote_usdt *=.*/scan_quote_usdt = true/' config.toml
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ grep -n 'scan_quote_usdt' config.toml
91:# không quy đổi/không trộn với WBNB trong 1 đường sim). scan_quote_usdt=false
96:scan_quote_usdt = true
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ git add config.toml && git commit -m "config: bật scan_quote_usdt cho 7 pool USDT đã vet"
[master 6b49943] config: bật scan_quote_usdt cho 7 pool USDT đã vet
 1 file changed, 1 insertion(+), 1 deletion(-)
dmin@DESKTOP-31M4TNT:~/bsc-sandwich$ scripts/paper_run.sh --minutes 5 --port 18799
== may chay: WSL (repo: /home/dmin/bsc-sandwich) ==
BSC_WS host = bsc-rpc.publicnode.com,wss (KHONG in URL day du)
  -> khop publicnode, OK
== cargo build --release ==
    Finished `release` profile [optimized] target(s) in 0.32s
== binary sha256 = 33f84ed709a658fb93d0bee2531ee39c5c718fbf74a565971bead0a9dec22bdf ==
== git HEAD = 6b49943defecda408cb1a4235e96af79e5b75322 ==
da xoa state/halt.lock (neu co)
== config TAM (chi nguong ve 0 + web_port, GIU NGUYEN sim_engine/pair_scan_universal/scan_quote_usdt tu config.toml that), port 18799 ==
== pairs.txt (grep tho, xem sim.evm/pair.unvetted trong log de co so that): tong=89 vetted=89 chua_vet=0 ==
== chay bot 5 phut, log -> logs/paper_run_1789474602.log (bot.jsonl tu dong 197388) ==
PID=1459134
```

Đọc log này:
- `89` / `91` / `2`: 89 dòng token, 2 dòng header cũng chứa chữ "vetted 2026" → khớp.
- Lệnh `awk … >2` không in gì → không dòng nào bị dính 2 địa chỉ.
- `77 insertions, 93 deletions`: thay toàn bộ thân file (93 dòng cũ ra, 77 + header vào).
- `tong=89 vetted=89 chua_vet=0` từ chính bot → bot đọc đúng file.
- `config.toml` dòng 96 `= true`, dòng 91 chỉ là comment — bình thường.

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

Quy tắc chung trong WSL, thư mục `~/bsc-sandwich`: sửa xong → xem đã đổi gì → thêm → commit → kiểm.

```bash
cd ~/bsc-sandwich
git status --short          # liệt kê file đã đổi (M = sửa, ?? = file mới)
git diff                    # xem nội dung đổi, nhấn q để thoát (bỏ qua nếu không cần)
git add <tên file>          # ví dụ: git add pairs.txt config.toml
git commit -m "mô tả ngắn: đổi gì, vì sao"
git log -1 --oneline        # thấy hash + message vừa commit
```

Ví dụ với lần vừa rồi (chỉ đổi thứ tự RPC trong `.env`): **không cần commit**, vì `.env` nằm trong `.gitignore` — `git status` sẽ không hiện nó. Đó là cố ý.

Muốn thêm tất cả file đã đổi một lượt: `git add -A` (nhớ nhìn `git status` trước để không lỡ thêm file rác như log). Muốn bỏ thay đổi của một file chưa commit: `git checkout -- <tên file>`.

Mẹo viết message để sau này (và auditor) đọc lại hiểu ngay: `pairs.txt: +13 meme PASS`, `config: bật scan_quote_usdt`, `.env.example: đổi thứ tự RPC, blxr xuống cuối`. Không cần dài.
