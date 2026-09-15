Được, và với danh sách vài trăm token thì check tay là cách hợp lý — miễn là bạn hiểu "an toàn cho sandwich" rộng hơn "không tax, không honeypot". Đây là checklist thực tế và cách làm đỡ tốn công.

**Điều kiện tiên quyết để check được**
- Contract phải **verified** trên BscScan (có source). Không verified → loại luôn, không cần nghĩ.
- Nếu là **proxy** (BscScan hiện tab "Read as Proxy") → owner có thể đổi toàn bộ logic bất cứ lúc nào → loại, trừ các stablecoin/blue-chip lớn (USDT, USDC, USD1 đều là proxy nhưng chấp nhận được vì rủi ro là "đóng băng" chứ không phải "đổi tax").

**Đọc source: tìm những thứ này (Ctrl+F trong tab Contract)**

| Tìm | Nghĩa | Kết luận |
|---|---|---|
| `fee`, `tax`, `_taxFee`, `liquidityFee`, `marketingWallet` | có cơ chế tax | Nếu tax hiện = 0 nhưng có `setFee`/`setTax` → owner đổi được → loại (hoặc chỉ nhận khi đã `renounceOwnership`) |
| `blacklist`, `isBlacklisted`, `_isBot`, `sniper`, `banned` | chặn địa chỉ | loại nếu owner còn quyền thêm |
| `maxTxAmount`, `maxWallet`, `_maxTx` | giới hạn giao dịch | front-run size bị chặn → phải biết số, hoặc loại |
| `cooldown`, `lastTrade`, `block.number`, `antiBot`, `launchBlock`, `tradingEnabled` | **anti-MEV** | **Quan trọng nhất cho sandwich**: token cấm mua và bán trong cùng block, hoặc cooldown vài giây → bot lỗ chắc. Loại |
| `pause`, `whenNotPaused`, `tradingOpen` | dừng giao dịch | loại nếu owner còn quyền |
| `mint` không giới hạn, `rebase`, `reflect`, `_rOwned` | rebase/reflection | balance thay đổi ngầm → sim sai → loại |
| `onlyOwner` + `owner()` ≠ `0x000…dEaD` | owner còn quyền | mọi thứ trên đều có thể bật lại sau |

**Tự động hóa 80% công việc trước khi đọc tay**
- GoPlus Token Security API (miễn phí, chain 56, batch nhiều địa chỉ): trả `buy_tax`, `sell_tax`, `is_honeypot`, `is_blacklisted`, `is_proxy`, `is_mintable`, `can_take_back_ownership`, `trading_cooldown`, `transfer_pausable`, `anti_whale`, `owner_change_balance`. Loại thẳng token nào có cờ xấu, chỉ đọc tay phần còn lại.
- honeypot.is và tokensniffer.com để đối chiếu thêm 1 nguồn.
- `measure_tax_evm` của chính bot (đã có) để xác nhận số buy/sell bps thật tại block hiện tại.

**Điều kiện pool**
- Pair V2 với WBNB hoặc USDT, `reserve_quote ≥ 20 BNB` (hoặc ≥ 10.000 USDT), và **là pool có volume thật** (mở BscScan pair, tab Transactions, thấy swap đều đặn). Pool lớn mà không ai swap thì không có victim.

**Sau khi vet tay, vẫn cần 2 thứ tự động**
1. Bot vet lại định kỳ (10 phút) bằng `measure_tax_evm` — bắt trường hợp owner bật tax/blacklist sau khi bạn check.
2. Live: đo lại đúng token đó ngay trước khi ký.

Gợi ý định dạng `pairs.txt` để ghi kết quả check tay, bot đọc được:
```
0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82 # CAKE | vetted 2026-09-15 | tax 0/0 | owner renounced | no cooldown
```

Với 100 token hiện tại, GoPlus lọc trong 5 phút, đọc tay phần còn lại (ước 30–40 token) khoảng 2–3 giờ. Bạn muốn mình viết script GoPlus batch cho `pairs.txt` để đưa vào lệnh cụm 2 không?
