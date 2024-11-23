root@disk-test-0:/# fio --section=test8 jobs.fio
test9: (g=0): rw=randwrite, bs=(R) 256KiB-256KiB, (W) 256KiB-256KiB, (T) 256KiB-256KiB, ioengine=libaio, iodepth=16
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [w(1)][100.0%][w=106MiB/s][w=424 IOPS][eta 00m:00s]
test9: (groupid=0, jobs=1): err= 0: pid=1431: Wed Feb 27 18:04:11 2019
  write: IOPS=784, BW=196MiB/s (206MB/s)(10.0GiB/52189msec); 0 zone resets
    slat (usec): min=88, max=3200.2k, avg=1210.52, stdev=16143.60
    clat (usec): min=2, max=3235.0k, avg=18224.70, stdev=62800.13
     lat (usec): min=126, max=3239.5k, avg=19435.93, stdev=64999.30
    clat percentiles (usec):
     |  1.00th=[   1991],  5.00th=[   2024], 10.00th=[   2040],
     | 20.00th=[   2089], 30.00th=[   2147], 40.00th=[  11994],
     | 50.00th=[  23987], 60.00th=[  23987], 70.00th=[  23987],
     | 80.00th=[  24249], 90.00th=[  32375], 95.00th=[  39584],
     | 99.00th=[  47973], 99.50th=[  48497], 99.90th=[  63177],
     | 99.95th=[  63701], 99.99th=[3238003]
   bw (  KiB/s): min=81920, max=1837568, per=100.00%, avg=222524.26, stdev=345396.12, samples=94
   iops        : min=  320, max= 7178, avg=869.21, stdev=1349.21, samples=94
  lat (usec)   : 4=0.01%, 250=0.01%, 500=0.01%
  lat (msec)   : 2=1.68%, 4=37.29%, 10=0.02%, 20=4.00%, 50=56.59%
  lat (msec)   : 100=0.38%
  cpu          : usr=0.38%, sys=16.29%, ctx=3637, majf=0, minf=1
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=100.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.1%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,40960,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=16

Run status group 0 (all jobs):
  WRITE: bw=196MiB/s (206MB/s), 196MiB/s-196MiB/s (206MB/s-206MB/s), io=10.0GiB (10.7GB), run=52189-52189msec

Disk stats (read/write):
  sdc: ios=0/13877, merge=0/27, ticks=0/47074892, in_queue=49468412, util=92.56%