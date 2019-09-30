# 一些关于 ipc 的性能数据

来源：https://stackoverflow.com/questions/1235958/ipc-performance-named-pipe-vs-socket

System: Linux (Linux ubuntu 4.4.0 x86_64 i7-6700K 4.00GHz)
Message: 128 bytes
Messages count: 1000000

#### Pipe benchmark:

```
Message size:       128
Message count:      1000000
Total duration:     27367.454 ms
Average duration:   27.319 us
Minimum duration:   5.888 us
Maximum duration:   15763.712 us
Standard deviation: 26.664 us
Message rate:       36539 msg/s
```


#### FIFOs (named pipes) benchmark:

```
Message size:       128
Message count:      1000000
Total duration:     38100.093 ms
Average duration:   38.025 us
Minimum duration:   6.656 us
Maximum duration:   27415.040 us
Standard deviation: 91.614 us
Message rate:       26246 msg/s
```

#### Message Queue benchmark:

```
Message size:       128
Message count:      1000000
Total duration:     14723.159 ms
Average duration:   14.675 us
Minimum duration:   3.840 us
Maximum duration:   17437.184 us
Standard deviation: 53.615 us
Message rate:       67920 msg/s
```

#### Shared Memory benchmark:

```
Message size:       128
Message count:      1000000
Total duration:     261.650 ms
Average duration:   0.238 us
Minimum duration:   0.000 us
Maximum duration:   10092.032 us
Standard deviation: 22.095 us
Message rate:       3821893 msg/s   <--
```

#### TCP sockets benchmark:

```
Message size:       128
Message count:      1000000
Total duration:     44477.257 ms
Average duration:   44.391 us
Minimum duration:   11.520 us
Maximum duration:   15863.296 us
Standard deviation: 44.905 us
Message rate:       22483 msg/s
```

#### Unix domain sockets benchmark:

```
Message size:       128
Message count:      1000000
Total duration:     24579.846 ms
Average duration:   24.531 us
Minimum duration:   2.560 us
Maximum duration:   15932.928 us
Standard deviation: 37.854 us
Message rate:       40683 msg/s
```

#### ZeroMQ benchmark:

```
Message size:       128
Message count:      1000000
Total duration:     64872.327 ms
Average duration:   64.808 us
Minimum duration:   23.552 us
Maximum duration:   16443.392 us
Standard deviation: 133.483 us
Message rate:       15414 msg/s
```