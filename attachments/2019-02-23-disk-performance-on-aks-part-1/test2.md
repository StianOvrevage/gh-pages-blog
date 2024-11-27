root@disk-test-0:/# fio --section=test2 jobs.fio
test2: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [W(1)][100.0%][w=121MiB/s][w=31.0k IOPS][eta 00m:00s]
test2: (groupid=0, jobs=1): err= 0: pid=1292: Wed Feb 27 17:09:56 2019
  write: IOPS=49.0k, BW=191MiB/s (201MB/s)(10.0GiB/53488msec); 0 zone resets
    slat (nsec): min=1900, max=2093.3M, avg=17262.35, stdev=1349544.91
    clat (nsec): min=600, max=2069.0k, avg=809.02, stdev=2566.21
     lat (usec): min=3, max=2093.3k, avg=18.50, stdev=1349.60
    clat percentiles (nsec):
     |  1.00th=[  604],  5.00th=[  604], 10.00th=[  700], 20.00th=[  700],
     | 30.00th=[  700], 40.00th=[  700], 50.00th=[  700], 60.00th=[  700],
     | 70.00th=[  804], 80.00th=[  804], 90.00th=[  804], 95.00th=[  804],
     | 99.00th=[ 1304], 99.50th=[ 2008], 99.90th=[15552], 99.95th=[28288],
     | 99.99th=[58624]
   bw (  KiB/s): min=26912, max=716872, per=100.00%, avg=208516.96, stdev=180253.06, samples=100
   iops        : min= 6728, max=179218, avg=52129.25, stdev=45063.25, samples=100
  lat (nsec)   : 750=60.68%, 1000=37.83%
  lat (usec)   : 2=0.97%, 4=0.17%, 10=0.04%, 20=0.23%, 50=0.07%
  lat (usec)   : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 4=0.01%
  cpu          : usr=6.47%, sys=25.22%, ctx=3357, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2621440,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=191MiB/s (201MB/s), 191MiB/s-191MiB/s (201MB/s-201MB/s), io=10.0GiB (10.7GB), run=53488-53488msec

Disk stats (read/write):
  sdc: ios=0/11720, merge=0/1, ticks=0/47240504, in_queue=49423376, util=90.07%