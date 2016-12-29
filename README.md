#dispatch_semaphore 信号量

###信号量概述（引用百度百科）：

　　以一个停车场的运作为例。简单起见，假设停车场只有三个车位，一开始三个车位都是空的。这时如果同时来了五辆车，看 门人允许其中三辆直接进入，然后放下车拦，剩下的车则必须在入口等待，此后来的车也都不得不在入口处等待。这时，有一辆车离开停车场，看门人得知后，打开 车拦，放入外面的一辆进去，如果又离开两辆，则又可以放入两辆，如此往复。在这个停车场系统中，车位是公共资源，每辆车好比一个线程，看门人起的就是信号量的作用。

抽象的来讲，信号量的特性如下：信号量是一个非负整数（车位数），所有通过它的线程/进程（车辆）都会将该整数减一（通过它当然是为了使用资源），当该整数值为零时，所有试图通过它的线程都将处于等待状态。在信号量上我们定义两种操作： Wait（等待） 和 Release（释放）。当一个线程调用Wait操作时，它要么得到资源然后将信号量减一，要么一直等下去（指放入阻塞队列），直到信号量大于等于一时。Release（释放）实际上是在信号量上执行加操作，对应于车辆离开停车场，该操作之所以叫做“释放”是因为释放了由信号量守护的资源。

###Demo解析 

####1
~~~
// 创建一个信号量，值为0  
dispatch_semaphore_t sema = dispatch_semaphore_create(0); 
// 在一个操作结束后发信号，这会使得信号量+1 
ABAddressBookRequestAccessWithCompletion(addressBook, ^(bool granted, CFErrorRef error) { 
     dispatch_semaphore_signal(sema);
});
 // 一开始执行到这里信号量为0，线程被阻塞，直到上述操作完成使信号量+1,线程解除阻塞 
dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
~~~

####2
~~~
// 创建一个组  
dispatch_group_t group = dispatch_group_create(); 
// 创建信号 信号量为10 
dispatch_semaphore_t semaphore = dispatch_semaphore_create(10); 
// 取得默认的全局并发队列 
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
for(int i = 0; i < 100; i++) { 
  // 由于信号量为10 队列里面最多会有10个人任务被执行，每次通过它信号量都减一，为0时就会等待 
  dispatch_semaphore_wait(semaphore,DISPATCH_TIME_FOREVER); 
  // 任务加到组内被监听 
  dispatch_group_async(group, queue, ^{ 
    NSLog(@"%i",i); 
    sleep(2); 
  //执行此行代码 信号量+1 如果为0 即0+1=1 上面wait语句又会放入一条任务 1-1=0 然后又陷入等待
  //即车满的停车场有一辆车离开 保安得知这个消息（信号量不为0）又开闸放了一辆车进来 （wait语句 1-1=0） 此时车又满了 又执行等待（信号量为0 执行到wait语句等待）
    dispatch_semaphore_signal(semaphore);
  }); 
} 
// 等待组内所有任务完成，否则阻塞 
dispatch_group_wait(group, DISPATCH_TIME_FOREVER); 
dispatch_release(group); //MRC
dispatch_release(semaphore); //MRC
~~~
