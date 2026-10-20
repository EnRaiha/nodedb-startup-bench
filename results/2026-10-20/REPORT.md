# NodeDB startup benchmark report

- **Run:** 2026-10-20, 06:10:54 to 06:21 UTC (about 10 minutes; the bundle README expects about 40 on a SATA SSD)
- **Bundle:** `nodedb-bench-comb2.zip`, release `v1` of `EnRaiha/nodedb-startup-bench`
- **Build under test:** NodeDB main `b7208babb` + startup bounds + startup logging (source `b264eb499`, binary reports `1a84e5bae`, binary sha256 `2024b725bff4`)
- **Plan:** 5 stores x 4 repeats x 2 backends = 40 boots, cold page cache (run as root)
- **Result:** **40 / 40 boots reached `serving`.** No `failed`, `timeout` or `exited` rows.

## Headline

| Store | WAL records | RAM (tmpfs) mean serving | Disk (ext4) mean serving | Disk / RAM |
| --- | ---: | ---: | ---: | ---: |
| `store-r5000` | 6,006 | 4.71 s | 4.98 s | 1.06x |
| `store-r15000` | 16,011 | 4.84 s | 6.94 s | 1.43x |
| `store-r30000` | 30,018 | 5.27 s | 12.29 s | 2.33x |
| `store-r60000` | 60,033 | 5.45 s | 24.39 s | 4.48x |
| `store-r120000` | 120,063 | 9.81 s | 46.56 s | 4.75x |

- RAM boots are nearly flat (4.7 to 5.5 s) from 6k to 60k records, then nearly double to 9.8 s at 120k.
- Disk boots grow with store size. Above 15k records the time is about proportional to record count, near 0.4 ms per record: 12.3 s, 24.4 s, 46.6 s at 30k, 60k, 120k.
- At the smallest store the two backends are indistinguishable (5.0 s vs 4.7 s). At the largest, disk is 4.75x slower than RAM.

## Time to `serving` (t_serving_s), 4 repeats per cell

| Backend | Store | n | min | median | mean | max | stdev | CV |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| ram | `store-r5000` | 4 | 4.01 | 4.88 | 4.71 | 5.09 | 0.50 | 10.7% |
| ram | `store-r15000` | 4 | 4.55 | 4.87 | 4.84 | 5.06 | 0.21 | 4.4% |
| ram | `store-r30000` | 4 | 4.93 | 5.27 | 5.27 | 5.60 | 0.29 | 5.5% |
| ram | `store-r60000` | 4 | 5.06 | 5.35 | 5.45 | 6.05 | 0.43 | 7.9% |
| ram | `store-r120000` | 4 | 9.11 | 9.90 | 9.81 | 10.33 | 0.52 | 5.3% |
| disk | `store-r5000` | 4 | 4.82 | 4.94 | 4.98 | 5.20 | 0.17 | 3.4% |
| disk | `store-r15000` | 4 | 5.87 | 7.07 | 6.94 | 7.74 | 0.93 | 13.5% |
| disk | `store-r30000` | 4 | 10.52 | 12.19 | 12.29 | 14.26 | 1.57 | 12.8% |
| disk | `store-r60000` | 4 | 23.14 | 24.38 | 24.39 | 25.68 | 1.07 | 4.4% |
| disk | `store-r120000` | 4 | 45.50 | 46.20 | 46.56 | 48.37 | 1.30 | 2.8% |

All times in seconds. `t_wall_s` (whole-process wall time) is on average 0.1 to 0.2 s above `t_serving_s` in every cell.

## WAL load time and I/O per boot (means)

| Backend | Store | t_wal_s | Read | Written |
| --- | --- | ---: | ---: | ---: |
| ram | `store-r5000` | 0.04 s | 16 MB | 0 MB |
| ram | `store-r15000` | 0.09 s | 0 MB | 0 MB |
| ram | `store-r30000` | 0.17 s | 0 MB | 0 MB |
| ram | `store-r60000` | 0.32 s | 0 MB | 0 MB |
| ram | `store-r120000` | 0.62 s | 0 MB | 0 MB |
| disk | `store-r5000` | 0.04 s | 85 MB | 192 MB |
| disk | `store-r15000` | 0.10 s | 105 MB | 559 MB |
| disk | `store-r30000` | 0.18 s | 130 MB | 1,286 MB |
| disk | `store-r60000` | 0.36 s | 179 MB | 2,362 MB |
| disk | `store-r120000` | 0.79 s | 285 MB | 6,991 MB |

- WAL load (`t_wal_s`) is 0.04 to 0.79 s: under 2% of a disk boot and at most about 6% of a RAM boot. It is not where most of the time goes.
- A disk boot writes far more than it reads: about 192 MB (smallest store) up to 6.99 GB (largest). The disk runs wrote **45.6 GB** in total, in line with the ~43 GB the bundle README expects.
- RAM boots wrote about 40 KB each. The one non-zero RAM read figure (16 MB mean on `store-r5000`) comes from the very first boot of the session (65.6 MB), not later ones.

## Disk fingerprint (fio, on the benchmarked disk vs tmpfs)

| Test | Disk (ext4 on virtio `vda`) | RAM (tmpfs) |
| --- | --- | --- |
| 4k write + fsync each write | 6 MiB/s, 1,571 IOPS, mean fsync 623.3 us | 2,074 MiB/s, 531,052 IOPS, mean fsync 0.2 us |
| 1 MiB seq write, direct, qd8 | 1,790 MiB/s, 1,790 IOPS | 281 MiB/s, 281 IOPS |
| 1 MiB seq read, direct, qd8 | 1,533 MiB/s, 1,533 IOPS | 4,376 MiB/s, 4,376 IOPS |
| 4k random read, direct, qd16 | 342 MiB/s, 87,673 IOPS, mean lat 182 us | 1,704 MiB/s, 436,233 IOPS, mean lat 36 us |

Direct I/O against tmpfs (the RAM seq-write figure in particular) is not a meaningful comparison, so read the RAM column as a reference only.

## Observations

1. **Everything served.** The 1-hour startup bounds in the bundle's `nodedb.toml` were never close to mattering: the slowest boot was 48.4 s, well under the 300 s Raft and 600 s data-group defaults.
2. **Disk cost scales with the store, RAM cost mostly does not.** The 4.75x disk/RAM gap at 120k records, against 1.06x at 6k, comes from the disk path: same binary, same CPU. The run does not show which part of the disk path is responsible. The disk boots average only about 150 MB/s of writes at the largest store, far below the 1.75 GiB/s sequential write fio measured, so raw bandwidth is not the limit. Per-write latency (fsync costs 0.62 ms here) is a candidate but was not tested.
3. **Write volume is the standout.** The largest store makes a disk boot write 6.99 GB, about 25x the 285 MB it reads. That write traffic is worth a look independent of the timings.
4. **Noise is moderate.** Run-to-run CV is 3% to 14% with only 4 repeats per cell. The two noisiest cells are disk `store-r15000` (13.5%, 5.87 / 6.45 / 7.74 / 7.70 s, which rises across repeats) and disk `store-r30000` (12.8%, 10.5 to 14.3 s). Treat differences smaller than about 15% in those cells as unresolved.
5. **Comparison with the bundle README's reference machine ("lotus", SATA SSD).** The README quotes 13 to 266 s for disk boots and 5 to 7 s for RAM boots. Here disk boots took 5.0 to 46.6 s and RAM boots 4.7 to 9.8 s. This VM's disk is much faster than that SATA SSD (about 1.75 GiB/s sequential write, 0.62 ms fsync), so the absolute numbers are not comparable and shouldn't be read as a regression or improvement.

## Caveats

- **Environment:** this ran in a cloud microVM (kernel `6.18.44-fc-v139`, 4 vCPU Intel Xeon @ 2.10 GHz, 15.7 GiB RAM, ext4 on a virtio disk). The bundle README says containers and similar environments are untested.
- **"Cold" means the guest page cache only.** `drop_caches` ran before each disk boot, but the hypervisor/host cache is outside this VM's control, which likely flatters the disk numbers.
- **Hardware metadata is blank.** Machine model, BIOS and CPU governor are empty and temperatures read 0 (no sensors in the VM). CPU frequency was a constant 2100 MHz at the start and end of every boot. Load average was 0.00 at the start of the session.
- **Sample size.** n = 4 per cell, so the means and standard deviations are indicative only.
- **Setup done by me:** I installed `zstd` (required), `fio` and `iproute2` (optional) with apt. Nothing in the bundle was modified. The bundle's SHA256SUMS all matched.
- The `nodedb-scan-probe` CPU probe measured 0.819 us per record per pass (2,447 MB/s, 4.92 ms per 12 MB pass).

## Appendix: every boot

| seq | backend | store | rep | records | t_wal_s | t_loop_s | t_gate_raft_s | t_gate_dg_s | t_serving_s | t_wall_s | outcome |
| ---: | --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| 1 | ram | `store-r5000` | 1 | 6,006 | 0.041 | - | 4.614 | 0.104 | 5.092 | 5.402 | serving |
| 2 | disk | `store-r5000` | 1 | 6,006 | 0.058 | - | 2.495 | 1.077 | 5.204 | 5.395 | serving |
| 3 | ram | `store-r5000` | 2 | 6,006 | 0.031 | - | 2.105 | 1.901 | 5.061 | 5.149 | serving |
| 4 | disk | `store-r5000` | 2 | 6,006 | 0.043 | - | 2.301 | 0.256 | 4.822 | 4.890 | serving |
| 5 | ram | `store-r5000` | 3 | 6,006 | 0.036 | - | 4.375 | 0.051 | 4.689 | 4.888 | serving |
| 6 | disk | `store-r5000` | 3 | 6,006 | 0.039 | - | 1.215 | 1.386 | 5.013 | 5.142 | serving |
| 7 | ram | `store-r5000` | 4 | 6,006 | 0.032 | - | 3.716 | 0.000 | 4.007 | 4.102 | serving |
| 8 | disk | `store-r5000` | 4 | 6,006 | 0.040 | - | 0.784 | 2.153 | 4.876 | 5.140 | serving |
| 9 | ram | `store-r15000` | 1 | 16,011 | 0.089 | - | 2.292 | 1.745 | 4.551 | 4.632 | serving |
| 10 | disk | `store-r15000` | 1 | 16,011 | 0.094 | - | 0.286 | 0.000 | 5.869 | 5.915 | serving |
| 11 | ram | `store-r15000` | 2 | 16,011 | 0.098 | - | 3.403 | 1.181 | 5.062 | 5.147 | serving |
| 12 | disk | `store-r15000` | 2 | 16,011 | 0.105 | - | 0.529 | 0.000 | 6.445 | 6.690 | serving |
| 13 | ram | `store-r15000` | 3 | 16,011 | 0.096 | - | 2.175 | 1.587 | 4.914 | 5.139 | serving |
| 14 | disk | `store-r15000` | 3 | 16,011 | 0.101 | - | 0.079 | 0.001 | 7.743 | 7.978 | serving |
| 15 | ram | `store-r15000` | 4 | 16,011 | 0.086 | - | 3.159 | 0.615 | 4.819 | 4.885 | serving |
| 16 | disk | `store-r15000` | 4 | 16,011 | 0.089 | - | 0.073 | 0.000 | 7.704 | 7.986 | serving |
| 17 | ram | `store-r30000` | 1 | 30,018 | 0.181 | - | 3.752 | 0.564 | 5.163 | 5.410 | serving |
| 18 | disk | `store-r30000` | 1 | 30,018 | 0.195 | - | 0.109 | 0.000 | 12.613 | 12.867 | serving |
| 19 | ram | `store-r30000` | 2 | 30,018 | 0.153 | - | 1.723 | 1.700 | 5.602 | 5.653 | serving |
| 20 | disk | `store-r30000` | 2 | 30,018 | 0.199 | - | 0.132 | 0.001 | 14.256 | 14.453 | serving |
| 21 | ram | `store-r30000` | 3 | 30,018 | 0.176 | - | 4.406 | 0.000 | 5.375 | 5.401 | serving |
| 22 | disk | `store-r30000` | 3 | 30,018 | 0.164 | - | 0.112 | 0.000 | 10.518 | 10.792 | serving |
| 23 | ram | `store-r30000` | 4 | 30,018 | 0.151 | - | 3.743 | 0.359 | 4.928 | 5.127 | serving |
| 24 | disk | `store-r30000` | 4 | 30,018 | 0.162 | - | 0.131 | 0.001 | 11.773 | 11.862 | serving |
| 25 | ram | `store-r60000` | 1 | 60,033 | 0.305 | - | 1.312 | 0.208 | 5.426 | 5.656 | serving |
| 26 | disk | `store-r60000` | 1 | 60,033 | 0.408 | - | 0.232 | 0.000 | 25.677 | 25.742 | serving |
| 27 | ram | `store-r60000` | 2 | 60,033 | 0.340 | - | 3.549 | 0.000 | 5.058 | 5.145 | serving |
| 28 | disk | `store-r60000` | 2 | 60,033 | 0.332 | - | 0.181 | 0.052 | 23.137 | 23.382 | serving |
| 29 | ram | `store-r60000` | 3 | 60,033 | 0.319 | - | 3.741 | 0.052 | 5.264 | 5.390 | serving |
| 30 | disk | `store-r60000` | 3 | 60,033 | 0.330 | - | 0.212 | 0.001 | 24.727 | 24.948 | serving |
| 31 | ram | `store-r60000` | 4 | 60,033 | 0.322 | - | 3.388 | 0.000 | 6.052 | 6.161 | serving |
| 32 | disk | `store-r60000` | 4 | 60,033 | 0.367 | - | 0.186 | 0.000 | 24.039 | 24.195 | serving |
| 33 | ram | `store-r120000` | 1 | 120,063 | 0.628 | - | 1.296 | 0.000 | 10.012 | 10.066 | serving |
| 34 | disk | `store-r120000` | 1 | 120,063 | 0.784 | - | 0.350 | 0.000 | 45.747 | 45.982 | serving |
| 35 | ram | `store-r120000` | 2 | 120,063 | 0.619 | - | 2.352 | 0.000 | 10.328 | 10.542 | serving |
| 36 | disk | `store-r120000` | 2 | 120,063 | 0.895 | - | 0.369 | 0.001 | 48.368 | 48.596 | serving |
| 37 | ram | `store-r120000` | 3 | 120,063 | 0.597 | - | 1.503 | 0.000 | 9.797 | 10.017 | serving |
| 38 | disk | `store-r120000` | 3 | 120,063 | 0.758 | - | 0.337 | 0.001 | 45.496 | 45.738 | serving |
| 39 | ram | `store-r120000` | 4 | 120,063 | 0.622 | - | 2.122 | 0.000 | 9.106 | 9.250 | serving |
| 40 | disk | `store-r120000` | 4 | 120,063 | 0.727 | - | 0.276 | 0.000 | 46.645 | 46.775 | serving |

`t_loop_s` is empty in every row of this run. The bundle does not document `t_gate_raft_s` or `t_gate_dg_s`, so they are shown as recorded and not interpreted here.

## Files

The full raw results (`runs.csv`, 40 boot logs, 1 Hz samples, fio JSON, `machine.txt`) are in `nodedb-bench-results-20261020-062108.tar.gz` (347 KB).
