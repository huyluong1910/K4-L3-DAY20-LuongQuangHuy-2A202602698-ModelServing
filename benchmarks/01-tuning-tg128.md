# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` Â· host `Windows-AMD64` Â· llama.cpp `b10488`
CPU: **4 physical Â· 8 logical** cores Â· `ngl=99` Â· metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 6.5 | 96% |
| 2 | 6.6 | 97% |
| 4 | 6.7 | 99% |
| 8 | 6.8 | 100% |
| 16 | 6.7 | 99% |

**Best**: `-t 8` at 6.8 tok/s
**Slowest tested**: `-t 1` at 6.5 tok/s (1.04x spread)
**Against the physical-core default** (`-t 4`, 6.7 tok/s): 1.01x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

Đường cong thông lượng decode (tg128) gần như phẳng trên dải quét từ 1 đến 16 threads (chỉ dao động từ 6.5 đến 6.8 tok/s, chênh lệch tối đa 1.04x).
Đỉnh đạt nhẹ ở -t 8 (8 logical cores) với 6.8 tok/s, và giảm nhẹ khi vượt quá số luồng logic ở -t 16 (6.7 tok/s) do chi phí context switching khi oversubscribe CPU.
Nguyên nhân cốt lõi khiến việc tăng số luồng (từ 1 lên 8 threads) không mang lại speedup đáng kể là vì pha **Decode (TPOT) bị nghẽn nghiêm trọng bởi băng thông bộ nhớ (memory bandwidth bound)** chứ không phải năng lực tính toán (FLOPs). Ở mỗi bước sinh ra 1 token, toàn bộ trọng số mô hình (gần 3 GB) đều phải được load từ RAM vào CPU cache. Vì băng thông bộ nhớ của máy có giới hạn vật lý và dùng chung cho toàn bộ các core, việc tăng thêm thread không thể tăng lượng bytes truyền tải từ RAM vào CPU, dẫn đến các luồng đều phải chờ dữ liệu từ bus bộ nhớ.
