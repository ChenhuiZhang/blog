---
title: RTC，系统时间，NTP
date: 2022-01-10 12:42:27
tags: kernel
---

最近遇到一个 RTC 同步的问题，记录下并梳理下具体同步的流程：通常情况下，操作系统启动后，会首先从 RTC 读取时间，这时的系统时间完全来自于 RTC 芯片；如果后续开启了 NTP 服务，系统里的 NTP 服务会和网络上的 NTP 服务器进行时间校准（一般会持续几分钟），一旦系统时间和 NTP 服务器时间同步完成后，操作系统会把当前的系统时间写回 RTC 芯片，并且在随后的时间里，定期的（Linux 上每隔 11 分钟）将系统时间写回 RTC。

这里涉及到三条时间轴，RTC，系统时间和 NTP， 我们假设 NTP 的时间线是标准的，那他们三者之间同步的话，一般依赖于以下几个因素：

<!-- more -->

- 每个时间数据能够提供的精度
- 每个时间线自身的误差
- 同步操作的时延

能提供的精度，系统时间和 NTP 可以认为是足够的，单位一般为纳秒，精确度则基本在毫秒级。短板是 RTC 芯片，通常的 RTC 芯片提供的接口只能到秒，所以无论读取还是写入，都只能到秒，毫秒的数值完全被忽略了。

时间线自身的误差，RTC 依赖于芯片的晶振，通常情况下是满足需求的（具体误差可以参看芯片手册），系统时间的误差同样依赖于主芯片的晶振，NTP 服务器的话同样类似。

考虑同步操作的时延，NTP 和系统之间通常是通过网络，这部分虽然时延很大并且波动大，但 NTP 协议本身就已经考虑到并通过算法补偿掉了；RTC 和系统时间的同步通常通过 i2c 协议进行传输，一般可以忽略。

所以整体来看，影响整体系统精度的最大短板就在于 RTC 能提供的精度，即使系统的时间精度很高，但写入 RTC 时，只能写到秒一级的数据，产生的误差一下子就有+/- 500ms 之多，同样道理，即使 RTC 本身的晶振精度很高，但系统刚起来去读时间的话，也只能读到秒，后面的毫秒数据完全没有，相当于又是- 1000ms 的误差。

所以如果想尽量让三者同步过程中保证获得一个高精度的数据，操作系统需要通过额外的一系列措施来实现，下面就通过代码梳理下系统是如何尽量补偿掉这个误差的。

涉及到的代码如下：

- openntpd-6.2p3

* linux-4.19

## NTP->系统时间->RTC

首先当 ntpd 服务完成时间调整时，会把 synced 标志设置为 1：

```c
int
ntpd_adjtime(double d)
{
  struct timeval  tv, olddelta;
  int   synced = 0;
  static int  firstadj = 1;

  d += getoffset();
  if (d >= (double)LOG_NEGLIGIBLE_ADJTIME / 1000 ||
      d <= -1 * (double)LOG_NEGLIGIBLE_ADJTIME / 1000)
    log_info("adjusting local clock. The current time diff is now %fs", d);
  else
    log_debug("adjusting local clock. The current time diff is now %fs", d);
  d_to_tv(d, &tv);
  if (adjtime(&tv, &olddelta) == -1)
    log_warn("adjtime failed");
  else if (!firstadj && olddelta.tv_sec == 0 && olddelta.tv_usec == 0)
    synced = 1;
  firstadj = 0;
  update_time_sync_status(synced);
  return (synced);
}
```

update_time_sync_status()会通过系统调用 adjtimex()把时间设置到 kernel 里：

```c
void
update_time_sync_status(int synced)
{
        struct timex txc = { 0 };

        txc.modes = MOD_STATUS;
        if (synced) {
                txc.modes |= MOD_MAXERROR;
                txc.maxerror = 0;
        } else
                txc.status = STA_UNSYNC;
        if (adjtimex(&txc) == -1)
                log_warn("ntp_adjtime (3) failed");
        return;
}
```

```c
int do_adjtimex(struct timex *txc)
{

   	... ...

  ntp_notify_cmos_timer();

  return ret;
}

void ntp_notify_cmos_timer(void)
{
  if (!ntp_synced()) // 此处会check STA_UNSYNC标志
    return;

  if (IS_ENABLED(CONFIG_GENERIC_CMOS_UPDATE) ||
      IS_ENABLED(CONFIG_RTC_SYSTOHC))
    queue_delayed_work(system_power_efficient_wq, &sync_work, 0);
}

/*
 * If we have an externally synchronized Linux clock, then update RTC clock
 * accordingly every ~11 minutes. Generally RTCs can only store second
 * precision, but many RTCs will adjust the phase of their second tick to
 * match the moment of update. This infrastructure arranges to call to the RTC
 * set at the correct moment to phase synchronize the RTC second tick over
 * with the kernel clock.
 */
static void sync_hw_clock(struct work_struct *work)
{

  if (!ntp_synced())
    return;

  if (sync_cmos_clock())
    return;

  sync_rtc_clock();
}

```

在 do_adjtimex()最后，会调用 ntp_notify_cmos_timer()，在 ntp_notify_cmos_timer()里 kernel 会启动一个 工作队列，并进入 sync_rtc_clock()

```c
static void sync_rtc_clock(void)
{
  unsigned long target_nsec;
  struct timespec64 adjust, now;
  int rc;

  if (!IS_ENABLED(CONFIG_RTC_SYSTOHC))
    return;

  getnstimeofday64(&now);

  adjust = now;
  if (persistent_clock_is_local)
    adjust.tv_sec -= (sys_tz.tz_minuteswest * 60);

  /*
   * The current RTC in use will provide the target_nsec it wants to be
   * called at, and does rtc_tv_nsec_ok internally.
   */
  rc = rtc_set_ntp_time(adjust, &target_nsec); //这里第一次写入RTC，然后根据返回值rc来决定下一次同步的间隔
  if (rc == -ENODEV)
    return;

  sched_sync_hw_clock(now, target_nsec, rc);
}
```

这里就涉及到之前提到的写入 RTC 产生的误差，kernel 会在这里做一个补偿，主要是通过多次写入来逼近。举例说明：假设系统获取到的当前时间为 m 秒 n 毫秒，第一次写入 RTC 时，只能先写入 m 秒，然后 kernel 计算出下一次写入的时间点是距离下一个整秒（m+1）的时间（1000 - n）毫秒，并通过定时器去触发下一次的写入操作。这样 1000-n 毫秒后，系统再一次去同步 RTC 时间，并写入 m+1 秒。这样 RTC 和系统的时间基本就到达一致了。

当然由于系统自身的误差（包括 CPU 调度的影响），有可能定时器触发的不是那么的精确，所以有可能需要再下一秒的同步，去设定时间（m+2）秒，通过这样不断的去逼近一个整秒的瞬间来对 RTC 进行同步。

```c
static void sched_sync_hw_clock(struct timespec64 now,
        unsigned long target_nsec, bool fail)

{
  struct timespec64 next;

  getnstimeofday64(&next);
  if (!fail)
    next.tv_sec = 659;
  else {
    /*
     * Try again as soon as possible. Delaying long periods
     * decreases the accuracy of the work queue timer. Due to this
     * the algorithm is very likely to require a short-sleep retry
     * after the above long sleep to synchronize ts_nsec.
     */
    next.tv_sec = 0;
  }

  /* Compute the needed delay that will get to tv_nsec == target_nsec */
  next.tv_nsec = target_nsec - next.tv_nsec;
  if (next.tv_nsec <= 0)
    next.tv_nsec += NSEC_PER_SEC;
  if (next.tv_nsec >= NSEC_PER_SEC) {
    next.tv_sec++;
    next.tv_nsec -= NSEC_PER_SEC;
  }

  queue_delayed_work(system_power_efficient_wq, &sync_work,
         timespec64_to_jiffies(&next));
}
```

还有一点每次写入 RTC 之前，kernel 会比较当前时间是不是已经非常接近一个整秒，

```c
 /* The ntp code must call this with the correct value in tv_nsec, if
  * it does not we update target_nsec and return EPROTO to make the ntp
  * code try again later.
  */
 ok = rtc_tv_nsec_ok(rtc->set_offset_nsec, &to_set, &now);
 if (!ok) {
   err = -EPROTO;
   goto out_close;
 }

```

rtc_tv_nsec_ok()会 check 当前时间（now）的 tv_nsec 是不是等于 rtc->set_offset_nsec+/-FUZZ

```c
  /* Allowed error in tv_nsec, arbitarily set to 5 jiffies in ns. */
  const unsigned long TIME_SET_NSEC_FUZZ = TICK_NSEC * 5;
```

如果满足误差范围，并且写入 RTC 成功，这一次的同步就完成了。kernel 会把下一次同步设定在 11 分钟后。

另一个需要注意的是，上面说的整秒其实不是默认行为，4.19 kernel 里的 rtc->set_offset_nsec，对应的就是想在哪个时间点同步 RTC，默认 rtc 驱动里设置为半秒，如果想做到整秒的同步，需要在驱动里把它改为 0。

## RTC->系统时间

反过来，当系统从断电状态启动时，kernel 会从 RTC 里读取时间，并作为系统时间。如果系统运行中没有网络或者没有 enable NTP 服务，那系统时间的精度完全依赖于从 RTC 里读取的时间。但不幸的是，读取 RTC 一般也只能读到秒，所以默认情况下，系统获取的时间都是不精确的。

```c
/* IMPORTANT: the RTC only stores whole seconds. It is arbitrary
 * whether it stores the most close value or the value with partial
 * seconds truncated. However, it is important that we use it to store
 * the truncated value. This is because otherwise it is necessary,
 * in an rtc sync function, to read both xtime.tv_sec and
 * xtime.tv_nsec. On some processors (i.e. ARM), an atomic read
 * of >32bits is not possible. So storing the most close value would
 * slow down the sync API. So here we have the truncated value and
 * the best guess is to add 0.5s.
 */

static int __init rtc_hctosys(void)
{
	int err = -ENODEV;
	struct rtc_time tm;
	struct timespec64 tv64 = {
		.tv_nsec = NSEC_PER_SEC >> 1,
	};
	struct rtc_device *rtc = rtc_class_open(CONFIG_RTC_HCTOSYS_DEVICE);

	if (rtc == NULL) {
		pr_info("unable to open rtc device (%s)\n",
			CONFIG_RTC_HCTOSYS_DEVICE);
		goto err_open;
	}

	err = rtc_read_time(rtc, &tm);
	if (err) {
		dev_err(rtc->dev.parent,
			"hctosys: unable to read the hardware clock\n");
		goto err_read;

	}

	tv64.tv_sec = rtc_tm_to_time64(&tm);

#if BITS_PER_LONG == 32
	if (tv64.tv_sec > INT_MAX) {
		err = -ERANGE;
		goto err_read;
	}
#endif

	err = do_settimeofday64(&tv64);

	dev_info(rtc->dev.parent,
		"setting system clock to "
		"%d-%02d-%02d %02d:%02d:%02d UTC (%lld)\n",
		tm.tm_year + 1900, tm.tm_mon + 1, tm.tm_mday,
		tm.tm_hour, tm.tm_min, tm.tm_sec,
		(long long) tv64.tv_sec);

err_read:
	rtc_class_close(rtc);

err_open:
	rtc_hctosys_ret = err;

	return err;
}
```

如上面代码里的那样，kernel 会默认填充 nsec 为半秒，也就是 500ms，所以系统时间在同步完后，已经有了+/- 500ms 的偏差。

如果未了同步时获得更高精度的时间，就需要额外的步骤来实现，通常有下面两种方法：

1. 类似 kernel 同步 RTC 时，采用不断逼近写入时间的方法，在我们从 RTC 读取时间时，我们也可以通过以固定的一个间隔来读取 RTC 时间，例如，假设我们间隔设为 d ms， 第一次读取时我们得到的时间是 T 秒，然后等待 d ms，再次读取 RTC，如果还是 T 秒，重复上述循环；如果读到 T+1 秒，那就退出循环，并立即同步系统时间为 T+1 秒，这样同步下来的精度可以达到 d ms （当然理论上需要加上系统定时器的精度）。
2. 通过利用 RTC 芯片上的中断管脚，有些 RTC 芯片的中断管脚上可以产生一个以 1 秒作为周期中断信号，这样 kernel 里可以在第一个中断到来时，读取下 RTC 时间，记为 T 秒，然后等到下一个中断到来时，立即同步系统时间为 T+1 秒（采用两个中断主要是规避读取 RTC 带来的一些耗时），这样同步的结果可以达到比较高的精度。

## 附录

下面是一次完整的系统从 NTP 同步，到系统同步到 RTC 的过程

```
May 27 09:42:14  systemd[1]: Started Network time.
May 27 09:42:14  kernel: do_adjtimex ...
May 27 09:42:14  kernel: do_adjtimex ...
May 27 09:42:14  ntpd[868]: ntp engine ready
May 27 09:42:14  kernel: do_adjtimex ...
May 27 09:42:15  kernel: do_adjtimex ...
May 27 09:42:15  ntpd[867]: set local clock to Thu May 27 09:42:15 UTC 2021 (offset 0.027322s)
May 27 09:42:22  kernel: do_adjtimex ...
May 27 09:42:22  kernel: do_adjtimex ...
May 27 09:42:31  kernel: do_adjtimex ...
May 27 09:42:31  kernel: do_adjtimex ...
May 27 09:42:37  kernel: do_adjtimex ...
May 27 09:42:37  ntpd[868]: peer 10.0.2.201 now valid
May 27 09:42:37  kernel: do_adjtimex ...
May 27 09:42:45  kernel: do_adjtimex ...
... ...
May 27 09:45:09  kernel: do_adjtimex ...
May 27 09:45:39  kernel: do_adjtimex ...
May 27 09:45:39  ntpd[868]: clock is now synced
May 27 09:45:39  kernel: do_adjtimex ...
May 27 09:45:39  kernel: do_adjtimex ...
May 27 09:45:39  kernel: do_adjtimex ...
May 27 09:45:39  kernel: do_adjtimex ...
May 27 09:45:39  kernel: do_adjtimex ...
May 27 09:45:39  kernel: pass ntp_synced
May 27 09:45:39  kernel: sync_hw_clock ...
May 27 09:45:39  kernel: write rtc time
May 27 09:45:39  kernel: tv_nsec: 887729612 (50000000)
May 27 09:45:39  kernel: now: 887729612, target: 0
May 27 09:45:39  kernel: sched_sync_hw_clock 0:111891499 target 0
May 27 09:45:40  kernel: sync_hw_clock ...
May 27 09:45:40  kernel: write rtc time
May 27 09:45:40  kernel: tv_nsec: 13155982 (50000000)
May 27 09:45:40  kernel: now: 13155982, target: 0
May 27 09:45:40  kernel: sched_sync_hw_clock 659:983555780 target 0
May 27 09:46:12  kernel: do_adjtimex ...
May 27 09:46:12  kernel: pass ntp_synced
May 27 09:46:13  kernel: do_adjtimex ...
May 27 09:46:13  kernel: pass ntp_synced
... ...
May 27 09:56:28  kernel: do_adjtimex ...
May 27 09:56:28  kernel: pass ntp_synced
May 27 09:56:44  kernel: sync_hw_clock ...
May 27 09:56:44  kernel: write rtc time
May 27 09:56:44  kernel: tv_nsec: 403066775 (50000000)
May 27 09:56:44  kernel: now: 403066775, target: 0
May 27 09:56:44  kernel: sched_sync_hw_clock 0:596448364 target 0
May 27 09:56:45  kernel: sync_hw_clock ...
May 27 09:56:45  kernel: write rtc time
May 27 09:56:45  kernel: tv_nsec: 12466498 (50000000)
May 27 09:56:45  kernel: now: 12466498, target: 0
May 27 09:56:45  kernel: sched_sync_hw_clock 659:983911530 target 0
```
