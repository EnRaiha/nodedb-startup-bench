# NodeDB startup benchmark

Boots a NodeDB build on synthetic stores and records how long each boot takes.

Build: main `b7208babb` + the startup bounds (300 s and 600 s defaults) + startup logging.

## Run

Needs Linux x86_64 with glibc 2.39 or newer. Download `nodedb-bench-comb2.zip` from the Releases page, then:

```
unzip nodedb-bench-comb2.zip
cd nodedb-bench-comb2
sudo ./run.sh
```

Without `sudo` the disk runs use a warm cache. Close heavy programs first and leave the machine alone until `send this file:` appears. Send back the `nodedb-bench-results-<time>.tar.gz` file that `run.sh` writes.

The zip holds its own binary, five synthetic stores and a `README.md` with the details. The plan runs 5 stores x 4 repeats x 2 backends and takes about 40 minutes on a SATA SSD.

NodeDB is licensed under the Business Source License 1.1. See `LICENSE`.
