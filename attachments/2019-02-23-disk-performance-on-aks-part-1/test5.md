root@disk-test-0:/# fio --section=test5 jobs.fio
test5: (g=0): rw=write, bs=(R) 128KiB-128KiB, (W) 128KiB-128KiB, (T) 128KiB-128KiB, ioengine=libaio, iodepth=1
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [W(1)][100.0%][w=24.6MiB/s][w=197 IOPS][eta 00m:00s]
test5: (groupid=0, jobs=1): err= 0: pid=1335: Wed Feb 27 17:31:12 2019
  write: IOPS=197, BW=24.7MiB/s (25.9MB/s)(7413MiB/300004msec); 0 zone resets
    slat (usec): min=18, max=2753, avg=45.59, stdev=20.98
    clat (usec): min=4377, max=32951, avg=5005.83, stdev=546.55
     lat (usec): min=4404, max=32978, avg=5052.52, stdev=547.89
    clat percentiles (usec):
     |  1.00th=[ 4555],  5.00th=[ 4686], 10.00th=[ 4686], 20.00th=[ 4752],
     | 30.00th=[ 4817], 40.00th=[ 4883], 50.00th=[ 4948], 60.00th=[ 4948],
     | 70.00th=[ 5014], 80.00th=[ 5080], 90.00th=[ 5211], 95.00th=[ 5538],
     | 99.00th=[ 7046], 99.50th=[ 8029], 99.90th=[ 9765], 99.95th=[10814],
     | 99.99th=[28443]
   bw (  KiB/s): min=20480, max=27136, per=99.99%, avg=25300.56, stdev=747.95, samples=600
   iops        : min=  160, max=  212, avg=197.64, stdev= 5.84, samples=600
  lat (msec)   : 10=99.91%, 20=0.07%, 50=0.02%
  cpu          : usr=0.47%, sys=1.16%, ctx=59340, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,59304,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=24.7MiB/s (25.9MB/s), 24.7MiB/s-24.7MiB/s (25.9MB/s-25.9MB/s), io=7413MiB (7773MB), run=300004-300004msec

Disk stats (read/write):
  sdc: ios=0/59300, merge=0/9, ticks=0/293716, in_queue=3584, util=1.20%