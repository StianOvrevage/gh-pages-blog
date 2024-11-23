root@disk-test-0:/# fio --section=test6 jobs.fio
test7: (g=0): rw=write, bs=(R) 256KiB-256KiB, (W) 256KiB-256KiB, (T) 256KiB-256KiB, ioengine=libaio, iodepth=1
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [W(1)][100.0%][w=120MiB/s][w=478 IOPS][eta 00m:00s]
test7: (groupid=0, jobs=1): err= 0: pid=1370: Wed Feb 27 17:40:56 2019
  write: IOPS=826, BW=207MiB/s (217MB/s)(10.0GiB/49558msec); 0 zone resets
    slat (usec): min=80, max=1955.2k, avg=1205.71, stdev=10251.65
    clat (nsec): min=800, max=112200, avg=1331.40, stdev=2065.15
     lat (usec): min=81, max=1955.2k, avg=1207.80, stdev=10251.98
    clat percentiles (nsec):
     |  1.00th=[  804],  5.00th=[  900], 10.00th=[  900], 20.00th=[  900],
     | 30.00th=[  900], 40.00th=[ 1004], 50.00th=[ 1004], 60.00th=[ 1004],
     | 70.00th=[ 1096], 80.00th=[ 1304], 90.00th=[ 2008], 95.00th=[ 2608],
     | 99.00th=[ 5408], 99.50th=[12608], 99.90th=[31872], 99.95th=[43264],
     | 99.99th=[68096]
   bw (  KiB/s): min=  512, max=1988608, per=100.00%, avg=216026.94, stdev=351154.76, samples=97
   iops        : min=    2, max= 7768, avg=843.84, stdev=1371.70, samples=97
  lat (nsec)   : 1000=36.23%
  lat (usec)   : 2=53.74%, 4=8.34%, 10=1.05%, 20=0.48%, 50=0.13%
  lat (usec)   : 100=0.03%, 250=0.01%
  cpu          : usr=0.48%, sys=11.64%, ctx=3419, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,40960,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=207MiB/s (217MB/s), 207MiB/s-207MiB/s (217MB/s-217MB/s), io=10.0GiB (10.7GB), run=49558-49558msec

Disk stats (read/write):
  sdc: ios=0/11700, merge=0/1, ticks=0/47025800, in_queue=49624932, util=97.19%