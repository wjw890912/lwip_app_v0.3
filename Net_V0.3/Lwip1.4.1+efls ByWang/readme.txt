


QQ:925295580
e-mail:925295580@qq.com
Time:201512
author:王均伟



       SW

   beta---(V0.1)


1、TCP/IP基础协议栈
.支持UDP
.支持TCP
.支持ICMP

2、超轻量级efs文件系统
.支持unix标准文件API
.支持SDHC标准。兼容V1.0和V2.0大容量SDK卡-16G卡无压力。（驱动部分参考开源 :-)）
.超低内存占用，仅用一个512字节的高速缓存来对文集遍历。

3、支持1-write DS18B20 	温度传感器
.支持单总线严格时序
.支持ROM搜索，遍历树叶子，允许一条总线挂接多个温度传感器
.数据自动转换为HTML文件

4、TCP/IP应用
.支持TFTP服务端，可以完成文件的下载任务。（此部分来自GITHUB，增加部分TIMEOUT 事件）tftp -i
.支持NETBIOS服务。
.支持一个TCP服务器，本地端口8080.	测试服务
.支持一个TCP客户端，本地端口40096	远端端口2301 远端IP192.168.0.163	用来做数据交互
.支持一个UDP广播，  本地端口02	    广播地址255.255.255.255	  用来把温度采集的数据发广播
.支持一个HTTP服务器 本地端口80  	http:ip  访问之	  关联2只18B20温度传感器显示在简单的SD卡的网页上

5、系统编译后
Program Size: Code=51264 RO-data=28056 RW-data=1712 ZI-data=55048  

6、网络配置

ipaddr：192, 168, 0, 253-209
netmask：255, 255, 0, 0
gw：192, 168, 0, 20
macaddress：xxxxxxxxxx


tks for GitHUB

	    HW

 stm32f107+DP83848CVV+ds18B20*2+SDHC card (4GB)

7、修改了一个SDcrad的BUG，有一个判断语句写错了，fix后可以支持2GB和4GB以上的两种卡片了。20160122
8、增加了一个宏在main.h中
#define MYSELFBOARD
如果定义了，那么表示使用李伟给我的开发板
如果不定义就选择我自己画的那块板子。
没有什么本质区别，框架一样，只是IO口稍有改动。20160122


 20160123
9、增加一个SMTP应用，可以通过定义USED_SMTP来使能 。
10、完成smtp.c的移植和测试。可以向我的邮箱发送邮件。邮箱需要设置关闭SSL。并且需要修改一下源码 的几个宏定义。
11、把采集的温度数据20分钟发一封邮件到我自己的邮箱中。完成。  必须用通过认真的IP否则过不去防火墙

12、调整了一下发邮箱的时间为5分钟一封邮件，@163邮箱的限制大约是300封邮件


20160402

13、测试中发现有内存泄露的情况,通过增加内存的信息通过TCP输出的 2301端口debug后发现
  异常的如下
 总内存数（字节）：6144
 已用内存（字节）：5816
 剩余内存数（字节）：328
 使用标示：1
 正常的如下
 总内存数（字节）：6144
 已用内存（字节）：20
 剩余内存数（字节）：6124
 使用标示：1
 显然memalloc使内存溢出查找代码因为除了SMTP应用程序使用malloc外其他不具有使用的情况。
 所以肯定是SMTP出问题
 进一步分析代码为SMTP的smtp_send_mail（）中
 smtp_send_mail_alloced(struct smtp_session *s)
 函数使用的
 s = (struct smtp_session *)SMTP_STATE_MALLOC((mem_size_t)mem_len);
 分配了一块内存没有事正常的释放。
 这样反复
 几次最终导致这块应用代码不能正常返回一块完整的 mem_le大小的内存块而一直保留了328字节的剩余内存。
 这最终导致了所有依赖mem的应用程序全部获取不到足够的内存块。而出现的内存溢出。
 继续分析 释放的内存句柄  (struct smtp_session *) s
 发现几处问题

 1）非正常中止	“风险”
   if (smtp_verify(s->to, s->to_len, 0) != ERR_OK) {
    return ERR_ARG;
  }
  if (smtp_verify(s->from, s->from_len, 0) != ERR_OK) {
    return ERR_ARG;
  }
  if (smtp_verify(s->subject, s->subject_len, 0) != ERR_OK) {
    return ERR_ARG;
  }
  由于没有对  smtp_send_mail_alloced 函数进行判断所以如果此处返回会造成函数不能正常中止
  也就会导致 (struct smtp_session *) s	没有机会释放（因为在不正常中止时是在后面处理的）
  但是考虑到源数据是固定的从片上flash中取得的，这种几率几乎没有。但是存在风险。所以统一改为
  if (smtp_verify(s->to, s->to_len, 0) != ERR_OK) {
    	 err = ERR_ARG;
     goto leave;
  }
  if (smtp_verify(s->from, s->from_len, 0) != ERR_OK) {
    	 err = ERR_ARG;
     goto leave;
  }
  if (smtp_verify(s->subject, s->subject_len, 0) != ERR_OK) {
    	 err = ERR_ARG;
     goto leave;
  }

  2）、非正常TCP连接，主要原因。
  原来的函数为：
  if(tcp_bind(pcb, IP_ADDR_ANY, SMTP_PORT)!=ERR_OK)
	{
	return	ERR_USEl;
  	   
	}
  显然还是同样的会造成malloc 分配了但是没有被调用，修改为
  if(tcp_bind(pcb, IP_ADDR_ANY,SMTP_PORT)!=ERR_OK)
  {
	err = ERR_USE;
    goto leave;		   
  }

   这样	  leave中就会自动处理释放掉这个非正常中止的而造成的内存的溢出问题。
   leave:
  smtp_free_struct(s);
  return err;

 归根结底是一个问题。那就是必须保证malloc 和free 成对出现。









