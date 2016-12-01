# Optimizing TLS over TCP to reduce latency for NGINX

* [`nginx__dynamic_tls_records`](https://github.com/cloudflare/sslconfig/blob/3e45b99/patches/)

### What we do now

We use a static record size of 4K.
This gives a good balance of latency and throughput.

#### Configuration

*Example*

```nginx
http {
  ssl_dyn_rec_enable on;
}
```

#### Optimize latency

By initialy sending small (1 TCP segment) sized records,
we are able to avoid HoL blocking of the first byte.
This means TTFB is sometime lower by a whole RTT.

#### Optimizing throughput

By sending increasingly larger records later in the connection,
when HoL is not a problem, we reduce the overhead of TLS record
(29 bytes per record with GCM/CHACHA-POLY).

#### Logic

Start each connection with small records
(1369 byte default, change with `ssl_dyn_rec_size_lo`).

After a given number of records (40, change with `ssl_dyn_rec_threshold`)
start sending larger records (4229, `ssl_dyn_rec_size_hi`).

Eventually after the same number of records,
start sending the largest records (`ssl_buffer_size`).

In case the connection idles for a given amount of time
(1s, `ssl_dyn_rec_timeout`), the process repeats itself
(i.e. begin sending small records again).

### Configuration directives

#### dyn_rec_enable
* **syntax**: `dyn_rec_enable bool`
* **default**: `off`
* **context**: `http`

#### dyn_rec_timeout
* **syntax**: `dyn_rec_timeout number`
* **default**: `1000`
* **context**: `http`

We want the initial records to fit into one TCP segment
so we don't get TCP HoL blocking due to TCP Slow Start.

A connection always starts with small records, but after
a given amount of records sent, we make the records larger
to reduce header overhead.

After a connection has idled for a given timeout, begin
the process from the start. The actual parameters are
configurable. If dyn_rec_timeout is 0, we assume dyn_rec is off.

#### dyn_rec_size_lo
* **syntax**: `dyn_rec_size_lo number`
* **default**: `1369`
* **context**: `http`

Default sizes for the dynamic record sizes are defined to fit maximal
TLS + IPv6 overhead in a single TCP segment for lo and 3 segments for hi:
1369 = 1500 - 40 (IP) - 20 (TCP) - 10 (Time) - 61 (Max TLS overhead)

#### dyn_rec_size_hi
* **syntax**: `dyn_rec_size_hi number`
* **default**: `4229`
* **context**: `http`

4229 = (1500 - 40 - 20 - 10) * 3  - 61

#### dyn_rec_threshold
* **syntax**: `dyn_rec_threshold number`
* **default**: `40`
* **context**: `http`

### License

* [Cloudflare](https://github.com/cloudflare), [Vlad Krasnov](https://github.com/vkrasnov)
