root@disk-test-0:/# fio --section=test4 jobs.fio
test4: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=16
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [W(1)][100.0%][w=4492KiB/s][w=1123 IOPS][eta 00m:00s]
test4: (groupid=0, jobs=1): err= 0: pid=1318: Wed Feb 27 17:24:14 2019
  write: IOPS=1122, BW=4488KiB/s (4596kB/s)(1315MiB/300027msec); 0 zone resets
    slat (usec): min=3, max=6057, avg=11.08, stdev=18.45
    clat (usec): min=1974, max=76123, avg=14244.79, stdev=18040.95
     lat (usec): min=1981, max=76139, avg=14256.51, stdev=18040.87
    clat percentiles (usec):
     |  1.00th=[ 2245],  5.00th=[ 2376], 10.00th=[ 2474], 20.00th=[ 2573],
     | 30.00th=[ 2671], 40.00th=[ 2737], 50.00th=[ 2835], 60.00th=[ 2999],
     | 70.00th=[ 5342], 80.00th=[41681], 90.00th=[43779], 95.00th=[44303],
     | 99.00th=[45351], 99.50th=[46400], 99.90th=[49021], 99.95th=[50070],
     | 99.99th=[55313]
   bw (  KiB/s): min= 4463, max= 4944, per=100.00%, avg=4488.26, stdev=18.83, samples=600
   iops        : min= 1115, max= 1236, avg=1122.04, stdev= 4.72, samples=600
  lat (msec)   : 2=0.01%, 4=67.38%, 10=4.04%, 20=0.06%, 50=28.46%
  lat (msec)   : 100=0.05%
  cpu          : usr=0.86%, sys=2.25%, ctx=120833, majf=0, minf=1
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=100.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.1%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,336655,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=16

Run status group 0 (all jobs):
  WRITE: bw=4488KiB/s (4596kB/s), 4488KiB/s-4488KiB/s (4596kB/s-4596kB/s), io=1315MiB (1379MB), run=300027-300027msec

Disk stats (read/write):
  sdc: ios=0/336544, merge=0/9, ticks=0/4770008, in_queue=4490192, util=100.00%