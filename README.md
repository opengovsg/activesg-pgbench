# ActiveSG pgbench

## Conclusions

Errors happen when all of the following conditions are met:

- querying through RDS Proxy
- with `--client` of more than what the proxy is allowed to make into the RDS cluster (50% more to make it very replicable), to force multiplexing on the proxy
- with `--protocol` of `prepared`, to use PostgreSQL extended query protocol

The errors in particular are:

```plaintext
ERROR:  portal "" does not exist
ERROR:  prepared statement "P_0" already exists
```

Which indicate that connections are sometimes mixed up by RDS Proxy when serving queries.

## Methodology

Tests are run against an Aurora PostgreSQL 15.6 regional cluster, with 1 * `db.t4g.medium` instance (max 394 connections allowed). The RDS Proxy is set up to take up to 100% of available DB connections (also 394).

`pgbench` is used to send `select-1.sql` (just `SELECT 1;`) repeatedly to the DB. That is the full test.

![Architecture](./architecture.png)

### ❌ Run with proxy on, high `--client`, `--protocol` of `prepared`, which causes errors

This is very replicable. More than 20 runs were made, a different number of errors appeared in every single one of them, at different stages of the runs.

💡 `--client` of 600 is chosen since 600 is 50% more than the max connection allowed on the DB instance itself (394 max conn), which forces proxy level multiplexing to happen, and triggers errors. Using too many more (e.g. 1000) makes borrowing connections too slow and the `pgbench` can never finish. Using only slightly more connections than the instance's max conn (e.g. 400, which is 6 more than 394) rarely triggers the errors since multiplexing doesn't happen frequently enough. `--protocol` of `prepared` instructs `pgbench` to use PostgreSQL extended query protocol. `--protocol` also supports `simple` which queries without the `prepare` statements.

For the other variables, `--jobs=10` means 10 threads which matches the no. of cores of the testing laptop, so it doesn't really matter. `--rate=100000` and `--time=20` just makes sure the test runs for 20 seconds at the maximum RPS possible.

```plaintext
➜ PGHOST='test-rds-proxy-d8c896a.proxy-clwpp2plrsoq.ap-southeast-1.rds.amazonaws.com' \
PGPORT=5432 \
PGUSER=postgres \
PGDATABASE=test \
pgbench \
  --client=600 \
  --file=./select-1.sql \
  --jobs=10 \
  --progress=1 \
  --protocol=prepared \
  --rate=100000 \
  --time=20

pgbench (16.4 (Homebrew), server 15.6)
starting vacuum...end.
progress: 5.0 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed, lag 0.000 ms
progress: 5.9 s, 582.5 tps, lat 884.963 ms stddev 18.258, 0 failed, lag 547.783 ms
progress: 6.2 s, 4101.5 tps, lat 1060.732 ms stddev 82.111, 0 failed, lag 932.697 ms
progress: 7.0 s, 4132.7 tps, lat 1608.082 ms stddev 233.602, 0 failed, lag 1436.444 ms
pgbench: pgbench:pgbench: error: client 114 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
error: client 52 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
 pgbench: pgbench: error: error: client 309 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
client 145 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
error: client 182 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 38 aborted in command 0 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 516 aborted in command 0 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 39 aborted in command 0 (SQL) of script 0; perhaps the backend died while processing
progress: 8.0 s, 2225.2 tps, lat 2318.087 ms stddev 240.158, 0 failed, lag 2167.663 ms
pgbench: error: client 382 aborted in command 0 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 585 aborted in command 0 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 147 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 251 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 429 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 523 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 55 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 278 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 534 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 399 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 100 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 204 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 15 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 425 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 247 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 144 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 427 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 453 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
progress: 9.0 s, 611.0 tps, lat 3581.591 ms stddev 273.086, 0 failed, lag 3098.952 ms
progress: 10.0 s, 1937.8 tps, lat 4605.857 ms stddev 221.121, 0 failed, lag 4452.187 ms
progress: 11.0 s, 2315.4 tps, lat 5427.768 ms stddev 293.991, 0 failed, lag 5312.041 ms
progress: 12.0 s, 3815.9 tps, lat 6454.173 ms stddev 263.546, 0 failed, lag 6014.152 ms
pgbench: error: client 556 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 78 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 542 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 527 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 149 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 358 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 229 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 394 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
progress: 13.0 s, 3443.0 tps, lat 7381.468 ms stddev 278.582, 0 failed, lag 7211.261 ms
progress: 14.0 s, 4017.0 tps, lat 8278.924 ms stddev 268.677, 0 failed, lag 8150.463 ms
pgbench: error: client 45 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 192 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 276 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 69 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 402 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 411 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 480 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 136 script 0 aborted in command 0 query 0: ERROR:  portal "" does not exist
pgbench: error: client 43 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 12 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 35 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 203 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 517 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 547 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 595 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
pgbench: error: client 296 script 0 aborted in command 0 query 0: ERROR:  prepared statement "P_0" already exists
progress: 15.0 s, 4463.5 tps, lat 9282.401 ms stddev 291.795, 0 failed, lag 9151.783 ms
progress: 16.2 s, 4364.9 tps, lat 10321.295 ms stddev 344.124, 0 failed, lag 10202.683 ms
progress: 17.0 s, 4745.6 tps, lat 11312.967 ms stddev 220.946, 0 failed, lag 11182.578 ms
progress: 18.0 s, 4381.1 tps, lat 12093.526 ms stddev 260.079, 0 failed, lag 11976.083 ms
progress: 19.0 s, 4877.7 tps, lat 13118.960 ms stddev 257.160, 0 failed, lag 13000.181 ms
progress: 20.0 s, 4453.9 tps, lat 14112.263 ms stddev 275.460, 0 failed, lag 13986.973 ms
transaction type: ./select-1.sql
scaling factor: 1
query mode: prepared
number of clients: 600
number of threads: 10
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 51186
number of failed transactions: 0 (0.000%)
latency average = 8699.600 ms
latency stddev = 3955.854 ms
rate limit schedule lag: avg 8537.507 (max 14600.129) ms
initial connection time = 4959.291 ms
tps = 3365.804033 (without initial connection time)
pgbench: error: Run was aborted; the above results are incomplete.
```

```plaintext
ERROR:  portal "" does not exist
ERROR:  prepared statement "P_0" already exists
```

The above errors indicate that multiplexing did not work properly during the run.

The next 3 examples will eliminate the variables one-by-one:

### ❌ Run with **proxy off**, high `--client`, `--protocol` of `prepared`, which fails as expected

⚠️ must make sure there is no existing connections on the DB before the run since the script consumes almost 100% of DB connections -> go to the instance, click `actions` -> `reboot`

The only difference is the `PGHOST`, which points to the cluster endpoint now. Notice the failures come directly from connection acquisition, since the run will need 600 connection which is more than the max no. of connections allowed:

```plaintext
➜ PGHOST='test-rds-cluster.cluster-clwpp2plrsoq.ap-southeast-1.rds.amazonaws.com' \
PGPORT=5432 \
PGUSER=postgres \
PGDATABASE=test \
pgbench \
  --client=600 \
  --file=./select-1.sql \
  --jobs=10 \
  --progress=1 \
  --protocol=prepared \
  --rate=100000 \
  --time=20

pgbench (16.4 (Homebrew), server 15.6)
starting vacuum...end.
pgbench: error: connection to server at "test-rds-cluster.cluster-clwpp2plrsoq.ap-southeast-1.rds.amazonaws.com" (13.228.86.138), port 5432 failed: FATAL:  remaining connection slots are reserved for non-replication superuser connections
pgbench: error: could not create connection for client 99
pgbench: error: connection to server at "test-rds-cluster.cluster-clwpp2plrsoq.ap-southeast-1.rds.amazonaws.com" (13.228.86.138), port 5432 failed: could not calculate client proof: OpenSSL failure
pgbench: error: could not create connection for client 158
```

Lowering the `--client` (to 380 in this case) does work:

```plaintext
➜ PGHOST='test-rds-cluster.cluster-clwpp2plrsoq.ap-southeast-1.rds.amazonaws.com' \
PGPORT=5432 \
PGUSER=postgres \
PGDATABASE=test \
pgbench \
  --client=380 \
  --file=./select-1.sql \
  --jobs=10 \
  --progress=1 \
  --protocol=prepared \
  --rate=100000 \
  --time=20

pgbench (16.4 (Homebrew), server 15.6)
starting vacuum...end.
progress: 2.5 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed, lag 0.000 ms
progress: 3.0 s, 9294.7 tps, lat 377.084 ms stddev 59.804, 0 failed, lag 347.816 ms
progress: 4.0 s, 15521.4 tps, lat 917.224 ms stddev 260.370, 0 failed, lag 892.971 ms
progress: 5.0 s, 18118.0 tps, lat 1751.926 ms stddev 233.748, 0 failed, lag 1730.879 ms
progress: 6.0 s, 16813.9 tps, lat 2578.875 ms stddev 246.893, 0 failed, lag 2556.334 ms
progress: 7.0 s, 17498.7 tps, lat 3370.431 ms stddev 231.551, 0 failed, lag 3348.786 ms
progress: 8.0 s, 16489.6 tps, lat 4255.720 ms stddev 234.832, 0 failed, lag 4232.532 ms
progress: 9.0 s, 18452.2 tps, lat 5045.577 ms stddev 235.746, 0 failed, lag 5025.020 ms
progress: 10.0 s, 17484.1 tps, lat 5871.997 ms stddev 239.389, 0 failed, lag 5850.387 ms
progress: 11.0 s, 12071.1 tps, lat 6716.171 ms stddev 266.033, 0 failed, lag 6684.677 ms
progress: 12.0 s, 12750.3 tps, lat 7591.038 ms stddev 267.660, 0 failed, lag 7561.138 ms
progress: 13.0 s, 17946.8 tps, lat 8436.475 ms stddev 238.359, 0 failed, lag 8415.335 ms
progress: 14.0 s, 14514.1 tps, lat 9288.917 ms stddev 272.009, 0 failed, lag 9262.646 ms
progress: 15.0 s, 17842.3 tps, lat 10097.065 ms stddev 229.710, 0 failed, lag 10076.135 ms
progress: 16.0 s, 13725.0 tps, lat 11022.550 ms stddev 243.953, 0 failed, lag 10994.499 ms
progress: 17.0 s, 15695.1 tps, lat 11814.861 ms stddev 265.354, 0 failed, lag 11790.678 ms
progress: 18.0 s, 12998.1 tps, lat 12620.650 ms stddev 267.235, 0 failed, lag 12591.619 ms
progress: 19.0 s, 13419.5 tps, lat 13522.608 ms stddev 242.221, 0 failed, lag 13494.184 ms
progress: 20.0 s, 13790.3 tps, lat 14448.729 ms stddev 257.689, 0 failed, lag 14421.073 ms
transaction type: ./select-1.sql
scaling factor: 1
query mode: prepared
number of clients: 380
number of threads: 10
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 270484
number of failed transactions: 0 (0.000%)
latency average = 7203.565 ms
latency stddev = 4177.118 ms
rate limit schedule lag: avg 7179.112 (max 14842.572) ms
initial connection time = 2472.277 ms
tps = 15382.399208 (without initial connection time)
```

Basically, with proxy off, when trying to acquire more connections than possible, connection acquisition errors are thrown and the runs are aborted, which is the expected behavior.

### ✅ Run with proxy on, **low enough `--client`**, `--protocol` of `prepared`, which passes

💡 `--client` of 394 is chosen to match the max conn allowed on the instance. Notice that technically some connections are still reused since RDS Proxy and Performance Insights etc. make a handful of connections to the DB all the time, reducing the usable connections to ~385. However, from testing, this tiny difference doesn't matter.

```plaintext
➜ PGHOST='test-rds-proxy-d8c896a.proxy-clwpp2plrsoq.ap-southeast-1.rds.amazonaws.com' \
PGPORT=5432 \
PGUSER=postgres \
PGDATABASE=test \
pgbench \
  --client=394 \
  --file=./select-1.sql \
  --jobs=10 \
  --progress=1 \
  --protocol=prepared \
  --rate=100000 \
  --time=20

pgbench (16.4 (Homebrew), server 15.6)
starting vacuum...end.
progress: 3.1 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed, lag 0.000 ms
progress: 4.0 s, 2039.4 tps, lat 674.571 ms stddev 120.237, 0 failed, lag 555.016 ms
progress: 5.0 s, 2869.8 tps, lat 1317.681 ms stddev 261.668, 0 failed, lag 1195.594 ms
progress: 6.0 s, 2659.6 tps, lat 2315.885 ms stddev 281.279, 0 failed, lag 2186.709 ms
progress: 7.0 s, 3263.3 tps, lat 3387.805 ms stddev 280.714, 0 failed, lag 3233.203 ms
progress: 8.0 s, 4270.1 tps, lat 4228.458 ms stddev 254.632, 0 failed, lag 4146.386 ms
progress: 9.0 s, 4370.1 tps, lat 5212.040 ms stddev 279.147, 0 failed, lag 5111.327 ms
progress: 10.0 s, 4706.9 tps, lat 6154.394 ms stddev 304.281, 0 failed, lag 6075.737 ms
progress: 11.0 s, 4739.1 tps, lat 7116.463 ms stddev 279.485, 0 failed, lag 7031.011 ms
progress: 12.0 s, 4518.7 tps, lat 8085.660 ms stddev 295.800, 0 failed, lag 7996.127 ms
progress: 13.0 s, 5195.0 tps, lat 9034.719 ms stddev 278.713, 0 failed, lag 8959.584 ms
progress: 14.0 s, 5125.1 tps, lat 10029.184 ms stddev 281.345, 0 failed, lag 9952.173 ms
progress: 15.0 s, 5262.2 tps, lat 10936.451 ms stddev 269.165, 0 failed, lag 10864.587 ms
progress: 16.0 s, 4339.3 tps, lat 11861.396 ms stddev 296.009, 0 failed, lag 11769.701 ms
progress: 17.0 s, 4670.4 tps, lat 12766.257 ms stddev 268.416, 0 failed, lag 12679.843 ms
progress: 18.0 s, 5068.7 tps, lat 13806.771 ms stddev 246.030, 0 failed, lag 13731.642 ms
progress: 19.0 s, 4942.2 tps, lat 14763.730 ms stddev 261.191, 0 failed, lag 14681.846 ms
progress: 20.0 s, 4944.6 tps, lat 15713.846 ms stddev 275.116, 0 failed, lag 15634.330 ms
transaction type: ./select-1.sql
scaling factor: 1
query mode: prepared
number of clients: 394
number of threads: 10
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 73221
number of failed transactions: 0 (0.000%)
latency average = 8977.743 ms
latency stddev = 4366.098 ms
rate limit schedule lag: avg 8887.681 (max 16208.597) ms
initial connection time = 3109.276 ms
tps = 4270.124553 (without initial connection time)
```

### ✅ Run with proxy on, high `--client`, **`--protocol` of `simple`**, which passes

This is the proof that multiplexing **does** work. It only has problems when extended query protocol is used.

```plaintext
➜ PGHOST='test-rds-proxy-d8c896a.proxy-clwpp2plrsoq.ap-southeast-1.rds.amazonaws.com' \
PGPORT=5432 \
PGUSER=postgres \
PGDATABASE=test \
pgbench \
  --client=600 \
  --file=./select-1.sql \
  --jobs=10 \
  --progress=1 \
  --protocol=simple \  
  --rate=100000 \
  --time=20

pgbench (16.4 (Homebrew), server 15.6)
starting vacuum...end.
progress: 4.5 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed, lag 0.000 ms
progress: 5.0 s, 4492.4 tps, lat 243.318 ms stddev 133.944, 0 failed, lag 152.237 ms
progress: 6.0 s, 2631.4 tps, lat 889.069 ms stddev 291.820, 0 failed, lag 709.237 ms
progress: 7.0 s, 3375.1 tps, lat 2071.838 ms stddev 278.459, 0 failed, lag 1854.576 ms
progress: 8.0 s, 5353.2 tps, lat 2883.189 ms stddev 280.903, 0 failed, lag 2765.594 ms
progress: 9.0 s, 5505.7 tps, lat 3880.868 ms stddev 277.180, 0 failed, lag 3769.680 ms
progress: 10.0 s, 5659.8 tps, lat 4802.553 ms stddev 280.494, 0 failed, lag 4698.429 ms
progress: 11.1 s, 5791.1 tps, lat 5767.913 ms stddev 293.977, 0 failed, lag 5668.691 ms
progress: 12.1 s, 5152.6 tps, lat 6759.927 ms stddev 292.240, 0 failed, lag 6642.885 ms
progress: 13.1 s, 6086.1 tps, lat 7647.908 ms stddev 277.894, 0 failed, lag 7556.707 ms
progress: 14.0 s, 5288.0 tps, lat 8606.091 ms stddev 265.583, 0 failed, lag 8489.759 ms
progress: 15.0 s, 5835.4 tps, lat 9516.948 ms stddev 275.479, 0 failed, lag 9405.063 ms
progress: 16.2 s, 5276.3 tps, lat 10521.825 ms stddev 291.897, 0 failed, lag 10421.362 ms
progress: 17.0 s, 3786.0 tps, lat 11548.261 ms stddev 210.908, 0 failed, lag 11454.003 ms
progress: 18.0 s, 4570.2 tps, lat 12397.703 ms stddev 288.641, 0 failed, lag 12327.268 ms
progress: 19.0 s, 4203.8 tps, lat 13243.478 ms stddev 278.323, 0 failed, lag 13017.057 ms
progress: 20.0 s, 4146.8 tps, lat 14282.558 ms stddev 283.517, 0 failed, lag 14209.398 ms
progress: 21.0 s, 337.7 tps, lat 14828.974 ms stddev 82.590, 0 failed, lag 14715.277 ms
transaction type: ./select-1.sql
scaling factor: 1
query mode: simple
number of clients: 600
number of threads: 10
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 75982
number of failed transactions: 0 (0.000%)
latency average = 7524.633 ms
latency stddev = 3983.105 ms
rate limit schedule lag: avg 7397.366 (max 14831.131) ms
initial connection time = 4492.426 ms
tps = 4526.962991 (without initial connection time)
```

### Conclusions

With all the experiments put together:

- ❌ Run with proxy on, high `--client`, `--protocol` of `prepared`, which causes errors
- ❌ Run with **proxy off**, high `--client`, `--protocol` of `prepared`, which fails as expected
- ✅ Run with proxy on, **low enough `--client`**, `--protocol` of `prepared`, which passes
- ✅ Run with proxy on, high `--client`, **`--protocol` of `simple`**, which passes

We can conclude that "proxy on, high `--client`, `--protocol` of `prepared`" are the 3 elements that causes multiplexing problems. Please go back to the start of this document to read the conclusions again.
