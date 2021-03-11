BIOS：0xF0000~0xFFFFF BIOS ROM，从0盘0道0扇区把mbr加载到0x7c00

boot/mbr.S：
	1.利用int 0x10BIOS中断的6号，13号子功能完成清屏和打印字符串操作
	2.把字符串写到显存中
	3.从0盘0道2扇区把loder加载到0x900，加载方式：将读取扇区数，lba地址，读命令写入硬盘控制器端口寄存器，待硬盘准备好后，将数据从端口读入到0x900
	
boot/loader.S：
	1.构建gdt（代码段，数据段和栈段，显存段）
	2.进入保护模式：
		a.打开A20，关闭地址回绕
		b.用gdtr寄存器存储gdt的地址
		c.将cr0寄存器的pe（protection enable）位置1
		d.用jmp指令刷新流水线
	3.加载内核：从第9个扇区把kernel.bin加载到0x70000并解析elf，丢弃elf头和程序头，把段提取到内核映像0x1500处
	4.创建页表：
		a.创建页目录和页表，一页大小为4k，容纳1024个页表项，页目录占用一个页表，页表起始地址为0x100000	
		b.初始化页目录项
			b1.构建第0项和第768项，使它们指向同一个页表，保证分页前后loader的运行的虚拟地址和物理地址一致，实现虚拟地址的高3G以上的虚拟地址映射到低1G的物理地址。
			b2.初始化并固定第769~1022个目录项，使所有进程共享内核
	5.将gdt段基址和显存段+0xc0000000,重新加载gdt

lib/kernel/print.S：
	1.读取显卡的光标寄存器获取光标值，在栈中获取参数
	2.根据参数分别实现打印普通值，退格键，回车键等功能,命名为put_char函数
	3.实现put_str,put_int等函数
	
kernel/kernel.S：宏定义中断处理程序，将每个中断处理函数入口intr_entry_table重定向到新的idt_table，把系统调用入口syscall_handler重定向到syscall_table

kernel/interrupt.c：
	pic_init():初始化可编程中断控制器8259A，设置起始中断向量号为0x20，开启IR0的时钟中断，IR1的键盘中断，IR2的级联接口，IR14的硬盘中断
	idt_desc_init():创建并初始化中断描述符表，将idt表的元素指向中断处理程序入口intr_entry_table，把idt[0x80]指向syscall_handler，单独处理系统调用
	exception_init()：中断处理函数的注册，把idt_table元素初始化为general_intr_handler
	idt_init()：中断初始化工作，代码实现{
		1.idt_desc_init()，初始化中断描述符表
		2.注册中断处理函数
		3.pic_init()，初始化8259A
		4.lidt,用idtr寄存器加载idt
	}
	
device/timer.c：
	timer_init()：初始化可编程计数器8253，将中断信号频率提高，注册时钟的中断处理函数
	intr_timer_handler()：时钟的中断处理函数，如果进程的时间片用完了，则schedule(),否则时间片-1
	mtime_sleep(m_seconds)：休眠函数，若间隔的ticks不够就一直thread_yield()

kernel/init.c：
	init_all()：{
		idt_init()：初始化中断
		mem_init()：初始化内存管理系统
		thread_init()：初始化线程
		timer_init()：初始化8253
		console_init()：控制台初始化
		
	}

	
lib/kernel/bitmap：bitmap实现，操作bitmap->bits，0代表未分配，1代表已分配，位图地址安排在0xc009a000

kernel/memory.c：
	mem_pool_init()：初始化内存池，内存池地址设置在页表上方
	vaddr_get(pf,pg_cnt)：在虚拟内存池申请pg_cnt个虚拟页，返回虚拟页的起始地址
	vaddr_remove(pf,_vaddr,pg_cnt)，pg_cnt个虚拟页回收到虚拟内存池
	palloc(m_pool)：在物理内存池申请一个物理页，返回物理页的地址
	pfree(pg_phy_addr)：将物理地址回收到物理内存池
	page_table_add(_vaddr,_page_phyaddr)：在页表中添加虚拟地址和物理地址的映射，根据物理地址构建页表项，根据虚拟地址获取页表项地址，将页表项赋值到地址
	page_table_pte_remove(vaddr)：去掉页表的映射，根据虚拟地址获取pte地址，将p位置0
	malloc_page(pf,pg_cnt)：分配pg_cnt个页空间，返回虚拟地址，实现代码{
		1.vaddr_get(pf,pg_cnt)
		2.palloc(m_pool)
		3.page_table_add(_vaddr,_page_phyaddr)}
	mfree_page(pf,_vaddr,pg_cnt)：释放_vaddr为起始地址的cnt个页空间，实现代码{
		1.判断在哪个池
		2.pfree(pg_phy_addr)
		3.page_table_pte_remove(vaddr)
		4.vaddr_remove(pf, _vaddr, pg_cnt)
	}
	block_desc_init(desc_array)：初始化内存块描述符数组
	sys_malloc(size)：在内存池中申请size字节内存，代码实现{
		1.根据cur_thread->pgdir判断用哪个内存池
		2.若size > 1024,就分配页框，若小于1024，匹配出合适的block_size
		3.判断此内存块没有空余空间，则新分配一页作为arena，然后拆分成内存块，弹出空余链表的队首作为内存块
	}
	sys_free(ptr)：回收内存，代码实现{
		1.根据cur_thread->pgdir判断用哪个内存池
		2.根据块构建arena，根据arena->large判断，
		3.若是大于1024，则释放页，若小于1024，则把内存块加入到free_list,若都空余，则释放整个arena
	}
							
thread/thread.c：
	thread_start(name,prio,function,func_arg)：创建线程，实现代码{
		1.分配pcb内存
		2.init_thread(pthread,name,prio),初始化线程基本信息
		3.thread_create(pthread,function,func_arg),初始化线程栈，赋值待执行的函数和参数
		4.list_append(plist,elem),加入到就绪队列和全部队列
	}
	schedule()：任务调度，代码实现{
		1.获取当前线程的pcb
		2.判断线程的状态，如果是正常RUNNING状态，说明只是时间用完了，则将线程加入到就绪队列，重置时间片，修改线程状态为TASK_READY
		3.弹出就绪队列的第一个线程，修改线程状态为TASK_RUNNING，准备调度
		4.激活页表
		5.switch.S,备份当前线程的上下文（寄存器与栈），恢复下一个线程的上下文，执行ret操作，将栈顶的函数地址返回给cs：ip寄存器
	}
	thread_init()：初始化线程环境，代码实现{
		1.list_init()
		2.make_main_thread()
	}
	thread_block(stat)：阻塞线程，关闭中断，获取当前线程的pcb，修改线程状态为阻塞状态，任务调度，恢复中断状态
	thread_unblock(pthread)：解除线程阻塞，关闭中断，将线程加入就绪队列的前端，让其尽快调度，修改线程状态为TASK_READY，恢复中断状态
	thread_yield(void)：让出cpu，关闭中断，获取pcb，修改线程状态为TASK_READY，加入就绪队列，任务调度，恢复中断状态
	
thread/sync.c：
	lock_init(plock)：锁的初始化，把锁的信号量初始化为1，0代表锁被持有，1代表锁未被持有,初始化锁的等待队列
	sema_down(psema)：信号量down操作，关闭中断，判断信号量值，若值为0，则代表锁被持有，把线程加入锁的等待队列，并阻塞线程，待被唤醒后，信号量值减1，获得锁，最后恢复中断状态
	sema_up(psema)：信号量up操作，关闭中断，从等待队列弹出队首线程，解除阻塞，信号量加1，释放锁，恢复中断状态
	lock_acquire(plock)：获取锁，sema_down(psema)
	lock_release(plock)：释放锁，sema_up(psema)
	
device/keyboard.c：
	1.流程：8048键盘编码器 -> 8042键盘控制器 -> 8259A中断代理
	1.构建keymap，以通码为索引，kemap[][0]为单个字符，keymap[][1]为控制键+字符
	2.定义控制键状态，从键盘buffer寄存器端口获取扫描码，如果是断码，则转换为通码
	3.再判断如果是控制键，则把相应的控制键状态设置为true，如果是普通字符，就把用cur_var变量存储，最后到keymap中取出对应值，若键盘缓冲区未满，则加入缓冲区

device/ioqueue.c：实现环形缓冲区
	1.定义环形队列，生产者消费者模型
	2.消费者，如果缓冲区为空，则阻塞消费者线程，若不为空，则取出字节，读游标移到下一位，若有生产者，则唤醒
	3.生产者，如果缓冲区为满，则阻塞生产者线程，若不为满，则写入字节，写游标移到下一位，若有消费者，则唤醒
	
userprog/process.c：
	3特权级栈顶0xc0000000
	0特权级栈顶pcb+PG_SIZE
	start_process(filename_)：初始化用户进程上下文信息
	create_page_dir()：创建页目录表，在内核内存池申请页目录地址，先复制页目录项到用户的页目录表，再更新页目录地址
	page_dir_activate(p_thread)：激活页表，若为内核线程，则重置页表地址，如果是用户进程，则将用户进程的页目录表地址加载到cr3页目录寄存器
	process_activate(p_thread)：page_dir_activate(p_thread)激活页表,如果是用户进程，则更新0特权级栈地址
	create_user_vaddr_bitmap(user_prog)：创建用户进程虚拟地址位图
	process_execute(filename,name)：创建用户进程，实现代码{
		1.分配pcb内存
		2.init_thread(pthread,name,prio),初始化线程基本信息
		3.create_user_vaddr_bitmap(user_prog)，创建用户进程虚拟地址位图
		4.thread_create(pthread,function,func_arg),初始化线程栈，赋值待执行的函数和参数
		5.create_page_dir()，创建页目录表
		6.list_append(plist,elem),加入到就绪队列和全部队列
	}
	
lib/user/syscall.c：宏定义_syscall()为int 0x80
	write(fd,buf,count)：系统调用实现整体流程，write -> _syscall3(SYS_WRITE) -> int 0x80,ax -> 进入中断syscall_table[SYS_WRITE] -> sys_write

lib/stdio.c：
	vsprintf(str,format,ap)：找出格式中的%号，再根据后面的格式依次将参数转化为对应的值存储在str中
	printf(format,...)：vsprintf(str,format,ap),write(fd,buf,count)

device/ide.c：
	ide_init：硬盘数据结构初始化
	select_disk(hd)：选择主从盘，根据hd->dev_no判断主从盘，构建device变量写入device寄存器完成主从盘的选择
	select_sector(hd,lba,sec_cnt)：写入lba地址和扇区数，往lba寄存器和sector_count寄存器写入相应变量
	cmd_out(channel,cmd)：写入命令，往cmd寄存器写入cmd命令
	read_from_sector(hd,buf,sec_cnt)：从硬盘读入sec_cnt扇区到buf，从data寄存器读数据到buf
	write2sector(hd,buf,sec_cnt)：把buf中sec_cnt扇区的数据写入硬盘，往data寄存器写入buf的数据
	ide_read(hd,lba,buf,sec_cnt)：从硬盘读sec_cnt扇区到buf，代码实现{
		select_disk(hd)，选择主从盘
		select_sector(hd,lba,sec_cnt)，写入lba地址和扇区数
		cmd_out(channel,cmd)，写入读命令
		sema_down()，阻塞自己，等待磁盘完成响应工作后唤醒自己
		read_from_sector(hd,buf,sec_cnt)，从硬盘读入sec_cnt扇区到buf
	}
	ide_write(hd,lba,buf,sec_cnt)：把buf中sec_cnt扇区的数据写入硬盘，代码实现{
		select_disk(hd)，选择主从盘
		select_sector(hd,lba,sec_cnt)，写入lba地址和扇区数
		cmd_out(channel,cmd)，写入写命令
		write2sector(hd,buf,sec_cnt)，把buf中sec_cnt扇区的数据写入硬盘
		sema_down()，阻塞自己，等待磁盘完成响应工作后唤醒自己
	}

fs/fs.c：
	partition_format(part)：格式化分区，创建文件系统，代码实现{
		1.计算文件系统各元素占有的扇区
		2.初始化超级块，将元信息的基本信息赋值到超级块，并写入引导扇区之后
		3.初始化块位图，inode位图，inode数组，写入相应的lba地址
		4.初始化根目录，写入两个目录项"."，".."到数据区开始处
	}
	mount_partition(pelem,arg)：挂载分区，将文件系统的元信息读入到内存part中
	filesys_init()：初始化文件系统，代码实现{
		1.读入超级快，根据魔数判断是否有文件系统，没有则partition_format(part)创建
		2.挂载分区
		3.初始化文件表
	}
	path_parse(pathname,name_store)：将最上层路径名称解析出来
	path_depth_cnt(pathname)：返回路径深度,比如/a/b/c,深度为3
	search_file(pathname,searched_record)：搜索文件pathname，若找到则返回inode号，代码实现{
		1.解析路径，记录父目录，
		2.search_dir_entry在父目录里查找相应文件，若是目录，则更新父目录，循环查找，若是文件则返回
	}
	sys_open(pathname,flags)：代码实现{
		1.解析路径
		2.file_create或file_open
	}
	sys_close
	sys_write(fd,buf,count)：将buf中连续count个字节写入文件描述符fd，fd分为标准输出，管道，硬盘
	sys_unlink(pathname)：删除文件，代码实现{
		1.解析路径，定位inode_no，记录父目录，
		2.回收inode，删除父目录的目录项
	}
	sys_mkdir(pathname)：创建目录pathname，代码实现{
		1.解析路径，search_file，若未找到，则新建inode_no并构建inode
		2.初始化目录，写入".",".."目录项
		3.在父目录中写入目录项，更新inode同步到硬盘
	}
	sys_rmdir(pathname)：删除空目录，dir_remove
	
	
fs/inode.c：
	inode_locate(part,inode_no,inode_pos)：获取inode所在的扇区和扇区内的偏移量
	inode_sync(part,inode,io_buf)：将inode写入到分区part中
	inode_open(part,inode_no)：根据inode_no找出对应的inode，代码实现{
		1.先在缓冲区open_inodes里找，若找到返回
		2.若未找到则inode_locate(part,inode_no,inode_pos)定位inode，并读出inode，并加入缓冲区
	}
	inode_close(inode)：关闭inode，从缓冲区去除，并释放inode空间
	inode_delete(part,inode_no,io_buf)：删除inode，代码实现{
		inode_locate,定位到inode_pos的扇区，先读出inode，清0之后，再覆盖
	}
	inode_release(part,inode_no)：回收inode的数据块和inode本身，代码实现{
		1.回收inode的数据块，根据块地址计算出块位图索引，清0同步，
		2.回收inode位图和inode本身
	}

fs/file.c：
	get_free_slot_in_global()：从文件表获取一个空闲位
	pcb_fd_install(fd_idx)：安装文件描述符
	inode_bitmap_alloc(part)：分配一个inode_no
	block_bitmap_alloc(part)：分配一个块（扇区号）
	bitmap_sync(part,bit_idx,btmp_type)：将位图中bit_idx对应的一扇区同步到硬盘
	file_create(parent_dir,filename,flag)：创建文件，成功则返回fd，代码实现{
		1.在inode_bitmap中分配inode_no，构建inode，在文件表中申请空位，分配fd，最后填充文件表
		2.构建目录项，同步到父目录，
		3.将父目录inode，文件inode，文件inode_bitmap同步到硬盘，
		4.将文件inode添加到open_inodes
		5.安装文件描述符
		6.回滚
	}
	file_open(inode_no,flag)：根据inode_no调用inode_open定位inode，存入文件表，安装文件描述符并返回
	file_close
	file_write(file,buf,count)：把buf中的count个字节写入file，代码实现{
		1.判断文件i_sectors[0]是否为0，若等于0，则说明是第一次写，则新分配一个块，并赋值扇区地址
		2.根据文件大小，计算出需要的块数，分为3种情况，直接块够用，跨直接块，直接块不够用，根据情况分配地址
		3.每次ide_write写入一扇区，记录写入的数据偏移索引，写完为止
	}
	file_read(file,buf,count)：从文件file中读取count个字节写入buf，代码实现{
		1.根据文件偏移量计算出数据所在块的起始和终止索引
		1.根据file->fd_inode-i_sectors锁定块，根据三种情况读入到all_blocks
		3.每次ide_read读入一扇区，记录读入的数据偏移索引，读完为止
	}

fs/dir.c
	open_root_dir(part)：打开根目录
	dir_open(part,inode_no)：根据inode_no调用inode_open(part,inode_no)，找出inode，返回目录指针
	dir_close(dir)：inode_close(inode),释放目录空间
	search_dir_entry(part,pdir,name,dir_e)：在分区的pdir目录内找名位name的文件或目录，找到将目录项存入dir_e，代码实现{
		1.读出pdir->inode->i_sectors，存入到all_blocks
		2.在块中遍历目录项，找到存入dir_e
	}
	create_dir_entry(filename,inode_no,file_type,p_de)：构建目录项
	sync_dir_entry(parent_dir,p_de,io_buf)：将目录项p_de写入父目录parent_dir，代码实现{
		1.获取到parent_dir->inode,根据块指针索引到块
		2.遍历所有块找到目录项空位，分配块，并同步块位图，写入块
	}
	delete_dir_entry(part,pdir,inode_no,io_buf)：把分区part目录pdir中编号为inode_no的目录项删除 ，代码实现{
		1.根据inode定位块
		2.遍历块，找到目录项
		3.回收块位图，清空块地址，清空目录项并同步，更新inode同步
	}
	dir_remove(parent_dir,child_dir)：delete_dir_entry，inode_release
	
userprog/fork.c：
	copy_pcb_vaddrbitmap_stack0(child_thread,parent_thread)：将父进程的pcb、虚拟地址位图拷贝给子进程，代码实现{
		1.复制pcb所在的整个页,里面包含进程pcb信息及特级0极的栈,里面包含了返回地址, 然后再单独修改个别部分 
		2.复制父进程的虚拟地址池的位图
	}
	copy_body_stack3(child_thread, parent_thread, buf_page)：复制子进程的进程体(代码和数据)及用户栈，代码实现{
		1.根据位图定位父进程虚拟地址
		2.切换页表，申请子进程虚拟地址
		3.将父进程用户空间数据复制到子进程
	}
	build_child_stack(child_thread)：修改switch_to的返回地址为intr_exit,让子进程被调度时直接从中断返回到fork代码处
	copy_process(child_thread, parent_thread)：拷贝父进程本身所占资源给子进程，代码实现{
		copy_pcb_vaddrbitmap_stack0
		create_page_dir()
		copy_body_stack3
		build_child_stack
	}
	
shell/pipe.c：管道实现
	sys_pipe(pipefd[])：申请一页内核内存做环形缓冲区
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
