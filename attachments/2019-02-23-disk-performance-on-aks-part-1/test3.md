root@disk-test-0:/# fio --section=test3 jobs.fio
test3: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [W(1)][100.0%][w=1513KiB/s][w=378 IOPS][eta 00m:00s]
test3: (groupid=0, jobs=1): err= 0: pid=1302: Wed Feb 27 17:17:35 2019
  write: IOPS=378, BW=1514KiB/s (1550kB/s)(444MiB/300001msec); 0 zone resets
    slat (usec): min=8, max=2863, avg=29.61, stdev=19.45
    clat (usec): min=2043, max=47769, avg=2595.50, stdev=598.13
     lat (usec): min=2063, max=47788, avg=2626.12, stdev=598.83
    clat percentiles (usec):
     |  1.00th=[ 2180],  5.00th=[ 2278], 10.00th=[ 2311], 20.00th=[ 2376],
     | 30.00th=[ 2409], 40.00th=[ 2442], 50.00th=[ 2474], 60.00th=[ 2507],
     | 70.00th=[ 2540], 80.00th=[ 2606], 90.00th=[ 2802], 95.00th=[ 3326],
     | 99.00th=[ 5538], 99.50th=[ 6194], 99.90th=[ 7898], 99.95th=[ 8455],
     | 99.99th=[12256]
   bw (  KiB/s): min= 1016, max= 1672, per=100.00%, avg=1518.67, stdev=97.35, samples=597
   iops        : min=  254, max=  418, avg=379.66, stdev=24.34, samples=597
  lat (msec)   : 4=97.20%, 10=2.78%, 20=0.01%, 50=0.01%
  cpu          : usr=0.72%, sys=2.58%, ctx=113720, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,113541,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=1514KiB/s (1550kB/s), 1514KiB/s-1514KiB/s (1550kB/s-1550kB/s), io=444MiB (465MB), run=300001-300001msec

Disk stats (read/write):
  sdc: ios=0/113518, merge=0/9, ticks=0/288108, in_queue=2028, util=0.68%