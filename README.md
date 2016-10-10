# stm32f4DelayRun
stm32f407芯片的延时运行程序；可设定延时xxms之后调用某个返回值和参数均为void的函数；

如果该文档与代码可能会有些不相符，是由于文档未及时更新

延时运行方案：
**必要条件：32bit ms 定时；
**

1: 32Bit ms计时：最大计时周期可用49天；可实现49天内的ms级延时运行；需要处理第49天处的交接；
2: 32Bit s计时：最大计时周期49000天=134年；可实现134年内的s级延时运行；需要处理134年处的交接；（如果设备可以连续运行134年）
3：{s:ms}计时结构体：实现最大计时周期134年；实现134年内的ms级延时运行；
4：{min:ms}计时结构体：实现最大计时周期134年；实现134年内的ms级延时运行；
5: {hh:ms}
6: {dd:ms}  
7: {mm:ms}
	

本方案采用3: {s:ms}延时方案；

延时运行块结构体
1:
struct DelayRunVoidStruct
{
	void (*proc)(void);
	struct DelayRunStruct *next;
};

2:
struct DelayRunAnyStruct
{
	void (*Proc)(uint_8 argc uint8_t** argv);
	struct DelayRunAnyStruct *next;
}

API:
	int8_t ProcIns_t(void (*proc)(void), {s:ms}，uint8_t InsSig );
	作用：将延时运行程序插入运行队列；
	参数：	Proc 进程指针；
			{s:ms}运行时间点；
			InsSig:	0:如果有多个进程时间点相同，则该进程插在最前；-1：插在最后；1+；插在第1+位置上；如果同一时间点的进程数少于1+则插在最后一个位置上；
	int_t ProcIns_d(void (*proc)(void), {s:ms}，uint8_t InsSig);
	作用：将延时运行程序插入运行队列；
	参数：	Proc 进程指针；
			{s:ms}延时时间；
			InsSig:	0:如果有多个进程时间点相同，则该进程插在最前；-1：插在最后；1+；插在第1+位置上；如果同一时间点的进程数少于1+则插在最后一个位置上；
	返回值：-1：插入失败；0+：插入成功，插入位置；
	
	void DelayRunKernel(void);
	延时运行核心调度进程；在指定时间执行相应的进程；如果多个进程在同一时间点，则依次调用；如果插入进程耗时较长，则会影响到其它任务的执行；
	该核心进程如果放到高优先级的中断中可以实现准时运行；不受低优先的中断服务进程影响；但不保证调度的进程是中断安全的；用户需用实现中断安全的过程；
	该核心如果在非中段的循环执行中；不用考虑进程的中断安全性；但是会影响到时效性；
	注：不建议延时运行IO操作；IO操作会占用大量实现，影响原有的系统时间；
	
	void DelayRunTick(void);
	用于维持延时运行系统的心跳；如果使用ms定时，需要将该函数放置到1ms的定时器中断之中；
	
	void DelayRunReset(void);
	作用：清空延时运行队列；对延时运行系统重新初始化；
	int8_t GetDrcLock();
	获取资源锁，获取到锁会自动上锁；并返回0；否则返回-1；
	int8_t ReleaseDrcLock();
	释放资源锁
全局变量：
	DelayRunChain；		延时运行链；
	DrcLock；			//延时运行链的资源锁
API实现：
	ProcIns_t  ：检查资源锁的状态；如果获取到锁，则上锁；执行插入动作；否则返回插入失败；（只有在中断中执行kernel时候才会发生获取资源索超时）
		沿着延时运行链进行对比；当发现定时时间相同的点之后检测InsSig，如果为0则直接插入；-1则一直向后寻找第一个比该时间点大的进程，插入到该进程前边；如果是1+，则向后寻找与当前相同时间点的进程；插在第1+之后；
	ProgIns_t  : 方法与ProcIns_t 相同；
	DelayRunTick :将系统时间++；
	DelayRunReset:清空延时运行队列；
	DelayRunKernel:检查资源锁的状态；如果未获得退出；如果获取到锁，则上锁执行下面的过程；执行完毕后解锁；否则：无法获取资源；返回；
		获取当前时间；将当前时间与延时运行队列的第一个进程时间对比，当前时间大于等于运行时间，则从队列中摘除该进程，执行该进程，并在次检测队列中的第一个进程的运行时间；否则退出；
		
	GetDrcLock：检查资源锁的状态：如果上锁；返回-1；如果获取到锁，返回0；
	ReleaseDrcLock:解锁资源锁；
