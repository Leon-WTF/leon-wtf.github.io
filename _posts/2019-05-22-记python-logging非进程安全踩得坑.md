---
category: Python
tags: [python, concurrency]
---
### 背景 ###
有两个python进程A和B复用同一个logger创建模块，往同一个文件里写日志，用的是TimedRotatingFileHandler，并且每天午夜进行文件rollover，保留15天的文件
<!-- more -->
### 现象 ###
偶尔会发现某一天的日志里记录的时间是后一天的，并且只有几行

### 原因 ###
虽然官方文档中说logging handler提供的类都是多线程安全的，但并不是多进程安全的，通过分析源码发现事实也确实如此。logging模块利用handler来负责日志文件的rollover，下面以TimedRotatingFileHandler为例来看下它的rollover是如何实现的：
- 所有打log的函数都会在Handler类中调用handle函数，然后调用emit函数：
```python
    def handle(self, record):
        """
        Conditionally emit the specified logging record.

        Emission depends on filters which may have been added to the handler.
        Wrap the actual emission of the record with acquisition/release of
        the I/O thread lock. Returns whether the filter passed the record for
        emission.
        """
        rv = self.filter(record)
        if rv:
            self.acquire()
            try:
                self.emit(record)
            finally:
                self.release()
        return rv
```
- 在TimedRotatingFileHandler的父类BaseRotatingHandler中重载了emit函数：
```python
    def emit(self, record):
        """
        Emit a record.

        Output the record to the file, catering for rollover as described
        in doRollover().
        """
        try:
            if self.shouldRollover(record):
                self.doRollover()
            logging.FileHandler.emit(self, record)
        except (KeyboardInterrupt, SystemExit):
            raise
        except:
            self.handleError(record)
```
可以看到其中利用shouldRollover判断是否需要rollover，利用doRollover来执行rollover
- TimedRotatingFileHandler中实现了这两个函数：
```python
    def shouldRollover(self, record):
        """
        Determine if rollover should occur.

        record is not used, as we are just comparing times, but it is needed so
        the method signatures are the same
        """
        t = int(time.time())
        if t >= self.rolloverAt:
            return 1
        #print "No need to rollover: %d, %d" % (t, self.rolloverAt)
        return 0
      
    def doRollover(self):
        """
        do a rollover; in this case, a date/time stamp is appended to the filename
        when the rollover happens.  However, you want the file to be named for the
        start of the interval, not the current time.  If there is a backup count,
        then we have to get a list of matching filenames, sort them and remove
        the one with the oldest suffix.
        """
        if self.stream:
            self.stream.close()
            self.stream = None
        # get the time that this sequence started at and make it a TimeTuple
        currentTime = int(time.time())
        dstNow = time.localtime(currentTime)[-1]
        t = self.rolloverAt - self.interval
        if self.utc:
            timeTuple = time.gmtime(t)
        else:
            timeTuple = time.localtime(t)
            dstThen = timeTuple[-1]
            if dstNow != dstThen:
                if dstNow:
                    addend = 3600
                else:
                    addend = -3600
                timeTuple = time.localtime(t + addend)
        dfn = self.baseFilename + "." + time.strftime(self.suffix, timeTuple)
        if os.path.exists(dfn):
            os.remove(dfn)
        # Issue 18940: A file may not have been created if delay is True.
        if os.path.exists(self.baseFilename):
            os.rename(self.baseFilename, dfn)
        if self.backupCount > 0:
            for s in self.getFilesToDelete():
                os.remove(s)
        if not self.delay:
            self.stream = self._open()
        newRolloverAt = self.computeRollover(currentTime)
        while newRolloverAt <= currentTime:
            newRolloverAt = newRolloverAt + self.interval
        #If DST changes and midnight or weekly rollover, adjust for this.
        if (self.when == 'MIDNIGHT' or self.when.startswith('W')) and not self.utc:
            dstAtRollover = time.localtime(newRolloverAt)[-1]
            if dstNow != dstAtRollover:
                if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                    addend = -3600
                else:           # DST bows out before next rollover, so we need to add an hour
                    addend = 3600
                newRolloverAt += addend
        self.rolloverAt = newRolloverAt
    def computeRollover(self, currentTime):
        """
        Work out the rollover time based on the specified time.
        """
        result = currentTime + self.interval
        # If we are rolling over at midnight or weekly, then the interval is already known.
        # What we need to figure out is WHEN the next interval is.  In other words,
        # if you are rolling over at midnight, then your base interval is 1 day,
        # but you want to start that one day clock at midnight, not now.  So, we
        # have to fudge the rolloverAt value in order to trigger the first rollover
        # at the right time.  After that, the regular interval will take care of
        # the rest.  Note that this code doesn't care about leap seconds. :)
        if self.when == 'MIDNIGHT' or self.when.startswith('W'):
            # This could be done with less code, but I wanted it to be clear
            if self.utc:
                t = time.gmtime(currentTime)
            else:
                t = time.localtime(currentTime)
            currentHour = t[3]
            currentMinute = t[4]
            currentSecond = t[5]
            # r is the number of seconds left between now and midnight
            r = _MIDNIGHT - ((currentHour * 60 + currentMinute) * 60 +
                    currentSecond)
            result = currentTime + r
            # If we are rolling over on a certain day, add in the number of days until
            # the next rollover, but offset by 1 since we just calculated the time
            # until the next day starts.  There are three cases:
            # Case 1) The day to rollover is today; in this case, do nothing
            # Case 2) The day to rollover is further in the interval (i.e., today is
            #         day 2 (Wednesday) and rollover is on day 6 (Sunday).  Days to
            #         next rollover is simply 6 - 2 - 1, or 3.
            # Case 3) The day to rollover is behind us in the interval (i.e., today
            #         is day 5 (Saturday) and rollover is on day 3 (Thursday).
            #         Days to rollover is 6 - 5 + 3, or 4.  In this case, it's the
            #         number of days left in the current week (1) plus the number
            #         of days in the next week until the rollover day (3).
            # The calculations described in 2) and 3) above need to have a day added.
            # This is because the above time calculation takes us to midnight on this
            # day, i.e. the start of the next day.
            if self.when.startswith('W'):
                day = t[6] # 0 is Monday
                if day != self.dayOfWeek:
                    if day < self.dayOfWeek:
                        daysToWait = self.dayOfWeek - day
                    else:
                        daysToWait = 6 - day + self.dayOfWeek + 1
                    newRolloverAt = result + (daysToWait * (60 * 60 * 24))
                    if not self.utc:
                        dstNow = t[-1]
                        dstAtRollover = time.localtime(newRolloverAt)[-1]
                        if dstNow != dstAtRollover:
                            if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                                addend = -3600
                            else:           # DST bows out before next rollover, so we need to add an hour
                                addend = 3600
                            newRolloverAt += addend
                    result = newRolloverAt
        return result
```
大致的步骤如下：
1. 判断当前时间是否大于下一次需要rollover的时间（初始化的最后第一次计算），如果大于则进行rollover
2. 计算rollover之后的文件名dfn，如果文件已经存在则删除
3. 将当前文件baseFilename重命名为上一步计算出的文件名dfn
4. 重新打开一个baseFilename的stream来写日志
5. 重新计算下一次需要rollover的时间
在初始化中第一次计算下一次rollover时间时：
```python
   def __init__(self, filename, when='h', interval=1, backupCount=0, encoding=None, delay=False, utc=False):
        BaseRotatingHandler.__init__(self, filename, 'a', encoding, delay)
        self.when = when.upper()
        self.backupCount = backupCount
        self.utc = utc
        # Calculate the real rollover interval, which is just the number of
        # seconds between rollovers.  Also set the filename suffix used when
        # a rollover occurs.  Current 'when' events supported:
        # S - Seconds
        # M - Minutes
        # H - Hours
        # D - Days
        # midnight - roll over at midnight
        # W{0-6} - roll over on a certain day; 0 - Monday
        #
        # Case of the 'when' specifier is not important; lower or upper case
        # will work.
        if self.when == 'S':
            self.interval = 1 # one second
            self.suffix = "%Y-%m-%d_%H-%M-%S"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}-\d{2}$"
        elif self.when == 'M':
            self.interval = 60 # one minute
            self.suffix = "%Y-%m-%d_%H-%M"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}$"
        elif self.when == 'H':
            self.interval = 60 * 60 # one hour
            self.suffix = "%Y-%m-%d_%H"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}$"
        elif self.when == 'D' or self.when == 'MIDNIGHT':
            self.interval = 60 * 60 * 24 # one day
            self.suffix = "%Y-%m-%d"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}$"
        elif self.when.startswith('W'):
            self.interval = 60 * 60 * 24 * 7 # one week
            if len(self.when) != 2:
                raise ValueError("You must specify a day for weekly rollover from 0 to 6 (0 is Monday): %s" % self.when)
            if self.when[1] < '0' or self.when[1] > '6':
                raise ValueError("Invalid day specified for weekly rollover: %s" % self.when)
            self.dayOfWeek = int(self.when[1])
            self.suffix = "%Y-%m-%d"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}$"
        else:
            raise ValueError("Invalid rollover interval specified: %s" % self.when)

        self.extMatch = re.compile(self.extMatch)
        self.interval = self.interval * interval # multiply by units requested
        if os.path.exists(filename):
            t = os.stat(filename)[ST_MTIME]
        else:
            t = int(time.time())
        self.rolloverAt = self.computeRollover(t)
```
如果文件已经存在，则以文件的修改时间为基础计算，如果不存在，则以当前时间为基础

- 另外还需要注意的是，在需要rollover的时间点，进程必须是在运行状态，如果进程退出又进入的话会重新计算下一次需要rollover的时间

基于以上分析不难看出，如果进程A先进行了rollover，进程B又进行rollover，真正的日志文件就会先被重名然后被删除掉，只有B进程rollover前A进程新写入的日志内容被重命名保存了下来。另外，由于在利用StreamHandler写文件时只使用了线程可重入锁，所以只能保证线程安全，多进程情况下会有并发写入的问题。

### 解决方案 ###
重载TimedRotatingFileHandler类：
- 将baseFilename也设为带时间后缀的形式，取消文件重名操作，仅在需要rollover的时候关闭当前stream，打开一个新文件名的stream
- 用文件锁替换线程锁
代码如下：
```python
import os
import time
import codecs
import fcntl
from logging.handlers import TimedRotatingFileHandler


class MultiProcessSafeHandler(TimedRotatingFileHandler):
    def __init__(self, filename, when='h', interval=1, backup_count=0, encoding=None, utc=False):
        TimedRotatingFileHandler.__init__(self, filename, when, interval, backup_count, encoding, True, utc)
        self.current_file_name = self.get_new_file_name()
        self.lock_file = None

    def get_new_file_name(self):
        return self.baseFilename + "." + time.strftime(self.suffix, time.localtime())

    def should_rollover(self):
        if self.current_file_name != self.get_new_file_name():
            return True
        return False

    def do_rollover(self):
        if self.stream:
            self.stream.close()
            self.stream = None
        self.current_file_name = self.get_new_file_name()
        if self.backupCount > 0:
            for s in self.getFilesToDelete():
                os.remove(s)

    def _open(self):
        if self.encoding is None:
            stream = open(self.current_file_name, self.mode)
        else:
            stream = codecs.open(self.current_file_name, self.mode, self.encoding)
        return stream

    def acquire(self):
        self.lock_file = open(self.baseFilename + ".lock", "w")
        fcntl.lockf(self.lock_file, fcntl.LOCK_EX)

    def release(self):
        if self.lock_file:
            self.lock_file.close()
            self.lock_file = None
```