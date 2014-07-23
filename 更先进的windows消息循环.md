```c++_
BOOL bRet;

while( (bRet = GetMessage( &msg, hWnd, 0, 0 )) != 0)
{ 
    if (bRet == -1)
    {
        // handle the error and possibly exit
    }
    else
    {
        TranslateMessage(&msg); 
        DispatchMessage(&msg); 
    }
}
```
####以上的代码作为开头，我相信作为windows程序员，都是十分熟悉的一段代码。
#####没错，他就是windows的GUI程序中的消息循环！
#####今天，我要拿这段代码开刀。因为他实在占用了太多太多不该占用的资源了！
#####作为一个循环，他只能够处理windows的UI消息。对于锁，系统的消息通知一概不能处理。
#####在看了 博士的[代码](https://github.com/yirenyiai/avbot),居然让我发现了windows还有一个函数，可以进行UI消息的等待，并且也可以等待其他通知消息。这个函数就是今天我要介绍的重点！
## **MsgWaitForMultipleObjects** 和 **MsgWaitForMultipleObjectsEx **
#####把这两个函数作为一个函数对待。这也是微软这渣渣的API设计。
#####说道这个份上了，这两个函数到底如何进行消息等待呢！OK，现在就用代码说话吧！
```c++

```
