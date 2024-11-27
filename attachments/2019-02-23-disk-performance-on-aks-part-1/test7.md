root@disk-test-0:/# fio --section=test7 jobs.fio
test8: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [w(1)][100.0%][w=71.0MiB/s][w=18.4k IOPS][eta 00m:00s]
test8: (groupid=0, jobs=1): err= 0: pid=1394: Wed Feb 27 17:52:13 2019
  write: IOPS=8330, BW=32.5MiB/s (34.1MB/s)(9762MiB/300003msec); 0 zone resets
    slat (usec): min=2, max=1027.5k, avg=116.73, stdev=1955.96
    clat (nsec): min=600, max=2830.0k, avg=870.11, stdev=3243.87
     lat (usec): min=3, max=1027.5k, avg=118.07, stdev=1956.29
    clat percentiles (nsec):
     |  1.00th=[  604],  5.00th=[  604], 10.00th=[  700], 20.00th=[  700],
     | 30.00th=[  700], 40.00th=[  700], 50.00th=[  700], 60.00th=[  804],
     | 70.00th=[  804], 80.00th=[  804], 90.00th=[  804], 95.00th=[  900],
     | 99.00th=[ 2800], 99.50th=[ 5984], 99.90th=[18560], 99.95th=[31616],
     | 99.99th=[69120]
   bw (  KiB/s): min=  776, max=659600, per=100.00%, avg=33598.44, stdev=90163.49, samples=595
   iops        : min=  194, max=164900, avg=8399.59, stdev=22540.88, samples=595
  lat (nsec)   : 750=57.34%, 1000=38.27%
  lat (usec)   : 2=2.99%, 4=0.77%, 10=0.26%, 20=0.27%, 50=0.07%
  lat (usec)   : 100=0.02%, 250=0.01%, 500=0.01%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%
  cpu          : usr=1.37%, sys=5.10%, ctx=17663, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2499097,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=32.5MiB/s (34.1MB/s), 32.5MiB/s-32.5MiB/s (34.1MB/s-34.1MB/s), io=9762MiB (10.2GB), run=300003-300003msec

Disk stats (read/write):
  sdc: ios=0/329033, merge=0/338, ticks=0/309996948, in_queue=311911340, util=98.53%