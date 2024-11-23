root@disk-test-0:/# fio --section=test1 jobs.fio
test1: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [W(1)][100.0%][w=44.8MiB/s][w=11.5k IOPS][eta 00m:00s]
test1: (groupid=0, jobs=1): err= 0: pid=1268: Wed Feb 27 17:04:08 2019
  write: IOPS=6934, BW=27.1MiB/s (28.4MB/s)(8126MiB/300001msec); 0 zone resets
    slat (usec): min=4, max=4793, avg=12.10, stdev= 9.97
    clat (nsec): min=1500, max=108751M, avg=128180.68, stdev=75399229.70
     lat (usec): min=40, max=108751k, avg=140.85, stdev=75399.23
    clat percentiles (usec):
     |  1.00th=[   52],  5.00th=[   60], 10.00th=[   62], 20.00th=[   63],
     | 30.00th=[   64], 40.00th=[   67], 50.00th=[   69], 60.00th=[   70],
     | 70.00th=[   74], 80.00th=[   80], 90.00th=[   94], 95.00th=[  106],
     | 99.00th=[  174], 99.50th=[  237], 99.90th=[  644], 99.95th=[  988],
     | 99.99th=[ 3621]
   bw (  KiB/s): min= 1976, max=46680, per=100.00%, avg=43326.10, stdev=3867.68, samples=383
   iops        : min=  494, max=11670, avg=10831.51, stdev=966.95, samples=383
  lat (usec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=0.01%, 50=0.70%
  lat (usec)   : 100=91.98%, 250=6.86%, 500=0.30%, 750=0.07%, 1000=0.03%
  lat (msec)   : 2=0.03%, 4=0.01%, 10=0.01%, 20=0.01%, 50=0.01%
  lat (msec)   : 500=0.01%
  cpu          : usr=6.23%, sys=17.93%, ctx=2080268, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2080360,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=27.1MiB/s (28.4MB/s), 27.1MiB/s-27.1MiB/s (28.4MB/s-28.4MB/s), io=8126MiB (8521MB), run=300001-300001msec