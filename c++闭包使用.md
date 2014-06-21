##**c++哲学**
###1> 让语言保持最精简的状态，一切的实现都交给库。
###2> 把特性的选择权交给用户，只有使用的时候，才生效。
###3> 追求通用，专用为次。

##**1.闭包**
####*笔者前言：不支持的闭包的语言，都应该被逐渐淘汰。*
####1.1 什么是闭包
#####在计算机科学中，闭包（Closure）是词法闭包（Lexical Closure）的简称，是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。
####1.2 闭包基本代码流程
#####在c++中让闭包真正发生质的飞跃，应该就是 *std::bind,std::function* 这两个函数纳入为 c++ 标准库以后。否则只能借住 *boost::bing,boost::function* 。闭包主要是用来处理事件的回调。一下是利用闭包的一个流程:

####1.3 闭包的实现
#####1.3.1 仿函数
#####在STL中大量使用了仿函数，仿函数相对与普通函数来说，他可以实现状态的初始化，保存。事实上，利用的就是c++的重载操作符实现的。以下是一个仿函数实例:
```c++
// 一个管理着QQ群的管理类
class QQGroupListInfo
{
public:
	std::vector<GroupData> vecGroup;
	void operator()( web::RecvInfo* pRecv,CURLcode Code,Webqq& qq,curl_slist *slist);
	std::string hashP(std::string uin,std::string ptwebqq);
	std::string CreatePostData( Webqq & qq  );
	static const std::string m_staPostUrl;
	static const std::string m_staReferer;
	static const std::string m_staContenType;
};
// 当我们在这里发起一个请求
int main()
{
    ......
    QQGroupListInfo qqGroup;
	AsnyCurlRun( std::bind(&QQGroupListInfo::operator(),qqGroup,std::placeholders::_1)  );
	.....
}

```
#####1.3.2 LAMBDA 表达式
#####值得一提的是，LAMBDA属于C++11的新特性。也属于语法糖一类。

####1.3.3 利用*std::bind/boost::bind*
#####绑定一个回调函数，当事件处理完毕后，就执行绑定的函数，对比于C语言的函数指针，使用*std::bind/boost::bind*能够完全避免野指针的问题

####1.4 与闭包相处融洽的小伙伴
#####1.4.1 IOCP
#####IOCP相比与协程来说，IOCP需要更多的资源，在*windows*上，实现IOCP的接口很难一次就用对。相比与linux上的epoll（I/O多路复用），IOCP是真正意义上的异步。
####show you the code!
``` c++
// 1:向IOCP线程投递一个闭包
	class HttpService
	{
        .....	
		// 所有任务投递完毕。进行一个调用让 curl 对任务进行处理
		void AsnyCurlRun( std::function<void(RecvInfo*,CURLcode)> pCBFunc );
        ......
	};
// 2: 实现
	void HttpService::AsnyCurlRun( std::function<void(RecvInfo*,CURLcode)> pCBFunc )
	{
		lpQueueData pdata = new QueueData;
		pdata->m_pCBFunc = pCBFunc;
		pdata->m_vecCurl = m_vecCurl;
		m_vecCurl.clear();

		// 把任务投递到完成端口线程中
		DWORD dwCompleteBytes = 1;
		PostQueuedCompletionStatus( curl_global_do_something::GetCompleteHandle(),dwCompleteBytes,NULL,&pdata->m_ol);
	}

// 3:IOCP后台线程
		UINT WINAPI curl_global_do_something::CurlWorkThread(void* lpParameter)
	{
		while( true )
		{
			DWORD NumberBytes = 0;
			DWORD dwCompletionKey = 0;
			LPOVERLAPPED ol= nullptr;

			BOOL bResult = GetQueuedCompletionStatus( hCurlCompletePort,&NumberBytes,(PULONG_PTR)&dwCompletionKey,&ol,INFINITE);
			QueueData *lpQueueData = (QueueData*)(CONTAINING_RECORD(ol, web::QueueData, m_ol));
			if( !NumberBytes || !lpQueueData ) 
			{
				TRACE("接收到一个空的完成包，线程退出\n");
				return 0;
			}

			std::shared_ptr<QueueData> pQueueData(lpQueueData);		// 重新获取该闭包运算中的数据

			if( !bResult ) 
			{
				// 返回出错
				assert( false );
				TRACE("完成端口线程异常退出，错误代码是:%d\n",GetLastError());
				return -1;
			}

			switch( NumberBytes )
			{
			case 1: 
				HandleMission( pQueueData );
				break;
			case 2: 
				ContinueHandleMission(pQueueData);
				break;
			default:
				assert(false);
				TRACE("接收到一个未知完成包，线程退出");
				break;
			}
		}
		return 0;
	}
```
异步： YES
解耦： YES
就是这么简单！由内核触发的回调，性能还会低？我不信，show you the code !

#####1.4.2 协程
#####对于协程，本人从来没有用过。


#####最后，留一下实际利用闭包完美解决问题的例子（至少本人感觉目前来说是完美的）
#####使用select结合curl mutile的使用中，在接口描述中有这么一段(直至写作的时候，[CURL](http://curl.haxx.se/)版本已经去到了7.37.0了)
```c++
/** 
* CURL 对 curl_multi_fdset 接口的原文(请在注意该注释只适用于当前版本。如果版本迭代请重新查阅)
* When max_fd returns with -1, 
* you need to wait a while and then proceed and call curl_multi_perform anyway. 
* How long to wait? I would suggest 100 milliseconds at least, 
* but you may want to test it out in your own particular conditions to find a suitable value. 
**/
```

##### 当然！我们不能直接使用sleep这种愚蠢的方案。
##### 在介绍IOCP的机制下使用闭包的代码中，每当我们调用 *AsnyCurlRun* 的时候，IOCP线程被唤醒。重新获取 *QueueData* 结构体。进入*HandleMission*或者进入*ContinueHandleMission*的函数。以下，就是整个处理函数
```c++
void curl_global_do_something::HandleMission( std::shared_ptr<QueueData>  pQueueData ,bool bWait )
	{
		CURLM	*CurlM=nullptr;							// curl 并发接口。
		CurlM = curl_multi_init();
		if( !CurlM )
			return ;
		PushCurlHandle( &CurlM,pQueueData );						// 进行任务的投递
		CurlMonitorCode Code=OnCurlMonitor( &CurlM , bWait);
		if(Code==CURL_MONITOR_CONTINUE)
		{
			// 重新生成闭包处理
			lpQueueData pCurlClosureData=new QueueData;
			pCurlClosureData->m_pCBFunc = pQueueData->m_pCBFunc;
			pCurlClosureData->m_vecCurl = pQueueData->m_vecCurl;
			pCurlClosureData->m_pCurlM  = CurlM;
			// 把任务投递到完成端口线程中
			DWORD dwCompleteBytes = 2;
			bExit=CURL_MONITOR_OK;
			PostQueuedCompletionStatus( curl_global_do_something::GetCompleteHandle(),dwCompleteBytes,NULL,&pCurlClosureData->m_ol);
			return ;
		}
		else  if(Code==CURL_MONITOR_ERROR)
		{
			// 出错了。
			TRACE("CURL请求出错\n");
			return ;
		}
		else if(Code==CURL_MONITOR_EXIT)
		{
			// 程序请求退出，中断所有处理
			TRACE("程序退出，要求退出所有的请求");
			return ;
		}
		OnCurlRunComplete( &CurlM,pQueueData );
		curl_multi_cleanup(CurlM);
	}
```
#####从上面的代码可以出，每当CURL需要Sleep的时候，会重新生成一个闭包，该闭包会触发 *ContinueHandleMission* 函数,然后接着处理。然后，以下就是 *ContinueHandleMission*
```c++
	void curl_global_do_something::ContinueHandleMission(  std::shared_ptr<QueueData> pQueue )
	{
		CurlMonitorCode Code=OnCurlMonitor( &pQueue->m_pCurlM );
		if(Code==CURL_MONITOR_CONTINUE)
		{
			// 重新生成闭包处理
			lpQueueData pCurlClosureData=new QueueData;
			pCurlClosureData->m_pCBFunc = pQueue->m_pCBFunc;
			pCurlClosureData->m_vecCurl = pQueue->m_vecCurl;
			pCurlClosureData->m_pCurlM  = pQueue->m_pCurlM;
			// 把任务投递到完成端口线程中
			DWORD dwCompleteBytes = 2;
			bExit=CURL_MONITOR_OK;
			PostQueuedCompletionStatus( curl_global_do_something::GetCompleteHandle(),dwCompleteBytes,NULL,&pCurlClosureData->m_ol);
			return ;
		}
		else  if(Code==CURL_MONITOR_ERROR)
		{
			// 出错了。
			TRACE("CURL请求出错\n");
			return ;
		}
		else if(Code==CURL_MONITOR_EXIT)
		{
			// 程序请求退出，中断所有处理
			TRACE("程序退出，要求退出所有的请求");
			return ;
		}
		OnCurlRunComplete( &pQueue->m_pCurlM,pQueue);
		curl_multi_cleanup(pQueue->m_pCurlM);
	}
```
#####自从看了AV社区的闭包使用文章，给了我无穷大的代码发挥空间，以上是我个人的总结，希望大家可以有所收益。如果有什么纰漏，也可以发邮件通知到我
#####个人邮件: <font color='blue'>lushuwen_gz@foxmail.com</font>
