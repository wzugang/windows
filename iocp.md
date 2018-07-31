```
#include <winsock2.h> 
#include <windows.h> 
#include <stdio.h>
#define PORT 5150 
#define DATA_BUFSIZE 8192
#pragma comment(lib, "Ws2_32")
typedef struct                        //这个玩意就是灌数据，取数据的一个自定义数据结构
                                              //和那个wm_data差不了多少，不过就是老要塞一个OverLapped结构， 
{ 
   OVERLAPPED Overlapped; 
   WSABUF DataBuf; 
   CHAR Buffer[DATA_BUFSIZE];                     
   DWORD BytesSEND;                                 //发送字节数 
   DWORD BytesRECV;                                 
} PER_IO_OPERATION_DATA, * LPPER_IO_OPERATION_DATA;

typedef struct 
{ 
   SOCKET Socket; 
} PER_HANDLE_DATA, * LPPER_HANDLE_DATA;

DWORD WINAPI ServerWorkerThread(LPVOID CompletionPortID);

void main(void) 
{ 
   SOCKADDR_IN InternetAddr; 
   SOCKET Listen; 
   SOCKET Accept; 
   HANDLE CompletionPort; 
   SYSTEM_INFO SystemInfo; 
   LPPER_HANDLE_DATA PerHandleData; 
   LPPER_IO_OPERATION_DATA PerIoData; 
   int i; 
   DWORD RecvBytes; 
   DWORD Flags; 
   DWORD ThreadID; 
   WSADATA wsaData; 
   DWORD Ret;
   if ((Ret = WSAStartup(0x0202, &wsaData)) != 0) 
   { 
      printf("WSAStartup failed with error %d\n", Ret); 
      return; 
   }
   // 
   //完成端口的建立得搞2次，这是第一次调用，至于为什么？我问问你 
   // 
   if ((CompletionPort = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0)) == NULL) 
   { 
      printf( "CreateIoCompletionPort failed with error: %d\n", GetLastError()); 
      return;
   } 
   //老套子api，不谈也罢 
   GetSystemInfo(&SystemInfo); 
   
   //发现2个CPU，那就开个双倍的线程跑吧 
   for(i = 0; i < SystemInfo.dwNumberOfProcessors * 2; i++) 
   { 
      HANDLE ThreadHandle; 
      
      // 
      //完成端口挂到线程上面来了，就像管子把灌数据的和读数据的两头都连上了
// 
      if ((ThreadHandle = CreateThread(NULL, 0, ServerWorkerThread, CompletionPort, 0, &ThreadID)) == NULL)
      { 
         printf("CreateThread() failed with error %d\n", GetLastError()); 
         return; 
      }      
      CloseHandle(ThreadHandle);
   }
   // 
   //启动一个监听socket ，以下都是长长的交代 
   // 
   if ((Listen = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, 
      WSA_FLAG_OVERLAPPED)) == INVALID_SOCKET) 
   { 
      printf("WSASocket() failed with error %d\n", WSAGetLastError());
      return; 
   }
   InternetAddr.sin_family = AF_INET; 
   InternetAddr.sin_addr.s_addr = htonl(INADDR_ANY); 
   InternetAddr.sin_port = htons(PORT);
   if (bind(Listen, (PSOCKADDR) &InternetAddr, sizeof(InternetAddr)) == SOCKET_ERROR) 
   { 
      printf("bind() failed with error %d\n", WSAGetLastError()); 
      return; 
   }  
   if (listen(Listen, 5) == SOCKET_ERROR) 
   { 
      printf("listen() failed with error %d\n", WSAGetLastError()); 
      return; 
   }
   // 
   // 监听端口打开，就开始在这里循环，一有socket连上，WSAAccept就创建一个socket， 
   // 这个socket 又和完成端口联上, 
   // 
   // 嘿嘿，完成端口第二次调用那个createxxx函数，为什么，留给人思考思考可能更深刻， 
   // 反正这套路得来2次， 
   // 完成端口completionport和accept socket挂起来了， 
   // 
   while(TRUE) 
   {
    //主线程跑到这里就等啊等啊，但是线程却开工了， 
      if ((Accept = WSAAccept(Listen, NULL, NULL, NULL, 0)) == SOCKET_ERROR) 
      { 
         printf("WSAAccept() failed with error %d\n", WSAGetLastError()); 
         return; 
      } 
      
   //该函数从堆中分配一定数目的字节数.Win32内存管理器并不提供相互分开的局部和全局堆.提供这个函数只是为了与16位的Windows相兼容
   //
      if ((PerHandleData = (LPPER_HANDLE_DATA) GlobalAlloc(GPTR, sizeof(PER_HANDLE_DATA))) == NULL) 
      { 
         printf("GlobalAlloc() failed with error %d\n", GetLastError()); 
         return; 
      }
      
      PerHandleData->Socket = Accept;
      
      // 
     //把这头和完成端口completionPort连起来 
     //就像你把漏斗接到管子口上,开始要灌数据了 
     // 
      if (CreateIoCompletionPort((HANDLE) Accept, CompletionPort, (DWORD) PerHandleData, 0) == NULL) 
      { 
         printf("CreateIoCompletionPort failed with error %d\n", GetLastError()); 
         return; 
      } 
      
      // 
      //清管子的数据结构,准备往里面灌数据 
      // 
      if ((PerIoData = (LPPER_IO_OPERATION_DATA) GlobalAlloc(GPTR,sizeof(PER_IO_OPERATION_DATA))) == NULL) 
      { 
         printf("GlobalAlloc() failed with error %d\n", GetLastError()); 
         return; 
      }
   //用0填充一块内存区域
      ZeroMemory(&(PerIoData->Overlapped), sizeof(OVERLAPPED)); 
      PerIoData->BytesSEND = 0; 
      PerIoData->BytesRECV = 0; 
      PerIoData->DataBuf.len = DATA_BUFSIZE; 
      PerIoData->DataBuf.buf = PerIoData->Buffer;
      Flags = 0; 
      
      // 
      // accept接到了数据，就放到PerIoData中,而perIoData又通过线程中的函数取出, 
     // 
      if (WSARecv(Accept, &(PerIoData->DataBuf), 1, &RecvBytes, &Flags, &(PerIoData->Overlapped), NULL) == SOCKET_ERROR) 
      { 
         if (WSAGetLastError() != ERROR_IO_PENDING) 
         { 
            printf("WSARecv() failed with error %d\n", WSAGetLastError()); 
            return; 
         } 
      } 
   } 
}
// 
//线程一但调用，就老在里面循环， 
// 注意，传入的可是完成端口啊，就是靠它去取出管子中的数据 
// 
DWORD WINAPI ServerWorkerThread(LPVOID CompletionPortID) 
{ 
   HANDLE CompletionPort = (HANDLE) CompletionPortID; 
   
   DWORD BytesTransferred; 
   LPOVERLAPPED Overlapped; 
   LPPER_HANDLE_DATA PerHandleData; 
   LPPER_IO_OPERATION_DATA PerIoData;         
   DWORD SendBytes, RecvBytes; 
   DWORD Flags; 

   while(TRUE)
{ 
      // 
      //在这里检查完成端口部分的数据buf区，数据来了吗？ 
      // 这个函数参数要看说明， 
      // PerIoData 就是从管子流出来的数据, 
      //PerHandleData 也是从管子里取出的，是何时塞进来的， 
     //就是在建立第2次createIocompletionPort时 
    // 
      if (GetQueuedCompletionStatus(CompletionPort, &BytesTransferred, (LPDWORD)&PerHandleData, (LPOVERLAPPED *) &PerIoData, INFINITE) == 0) 
      { 
         printf("GetQueuedCompletionStatus failed with error %d\n", GetLastError()); 
         return 0; 
      }
      // 检查数据传送完了吗 
      if (BytesTransferred == 0) 
      { 
         printf("Closing socket %d\n", PerHandleData->Socket);
         if (closesocket(PerHandleData->Socket) == SOCKET_ERROR) 
         { 
            printf("closesocket() failed with error %d\n", WSAGetLastError()); 
            return 0; 
         }
         GlobalFree(PerHandleData); 
         GlobalFree(PerIoData); 
         continue; 
      }     
     // 
    //看看管子里面有数据来了吗？=0，那是刚收到数据 
    // 
      if (PerIoData->BytesRECV == 0) 
      { 
         PerIoData->BytesRECV = BytesTransferred; 
         PerIoData->BytesSEND = 0; 
      } 
      else   //来了， 
      { 
         PerIoData->BytesSEND += BytesTransferred; 
      } 
   printf("print: %d\n", PerIoData->BytesRECV); 
   
      // 
      // 数据没发完？继续send出去 
     // 
     if (PerIoData->BytesRECV > PerIoData->BytesSEND) 
      {
         ZeroMemory(&(PerIoData->Overlapped), sizeof(OVERLAPPED)); //清0为发送准备 
         PerIoData->DataBuf.buf = PerIoData->Buffer + PerIoData->BytesSEND; 
         PerIoData->DataBuf.len = PerIoData->BytesRECV - PerIoData->BytesSEND;
       //1个字节一个字节发送发送数据出去 
         if (WSASend(PerHandleData->Socket, &(PerIoData->DataBuf), 1, &SendBytes, 0, 
            &(PerIoData->Overlapped), NULL) == SOCKET_ERROR) 
         { 
            if (WSAGetLastError() != ERROR_IO_PENDING) 
            { 
               printf("WSASend() failed with error %d\n", WSAGetLastError()); 
               return 0; 
            } 
         } 
      } 
      else 
      { 
         PerIoData->BytesRECV = 0;
         Flags = 0; 
         ZeroMemory(&(PerIoData->Overlapped), sizeof(OVERLAPPED));
         PerIoData->DataBuf.len = DATA_BUFSIZE; 
         PerIoData->DataBuf.buf = PerIoData->Buffer;
         if (WSARecv(PerHandleData->Socket, &(PerIoData->DataBuf), 1, &RecvBytes, &Flags, 
            &(PerIoData->Overlapped), NULL) == SOCKET_ERROR) 
         { 
            if (WSAGetLastError() != ERROR_IO_PENDING) 
            { 
               printf("WSARecv() failed with error %d\n", WSAGetLastError()); 
               return 0; 
            } 
         } 
      } 
   } 
}

例子:http://www.codeproject.com/KB/IP/iocp_server_client/IOCP-Demo.zip

```
