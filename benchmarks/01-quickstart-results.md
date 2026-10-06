# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 64545 | 1309 / 12480 | 190.4 / 204.6 | 13400 / 23155 / 23155 | 5.3 |
| UD-Q2_K_XL | 2.24 | 25599 | 2265 / 19694 | 276.2 / 284.3 | 19671 / 36539 / 36539 | 3.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.47x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên máy tính CPU 4 physical cores không có discrete GPU offload đầy đủ, mô hình bị giới hạn bởi năng lực tính toán CPU (compute-bound) khi dequantize hơn là băng thông bộ nhớ (memory bandwidth).
Do đó, phiên bản UD-Q2_K_XL (2.24 GB) dù tiết kiệm được 0.73 GB RAM (~25% dung lượng) nhưng tốc độ decode lại chậm hơn 1.47x (3.6 tok/s so với 5.3 tok/s ở UD-Q4_K_XL). Chi phí unpack/dequantize các block 2-bit tốn nhiều chu kỳ CPU hơn lượng memory I/O tiết kiệm được. Đồng thời, chất lượng sinh câu trả lời của 2-bit suy giảm rõ rệt. Với máy có 15.7 GB RAM thoải mái chứa bản 4-bit, UD-Q2_K_XL hoàn toàn không đáng để đánh đổi.
