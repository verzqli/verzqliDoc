#### frameworks/native/libs/binder/  

- Binder.cpp  ()

- BpBinder.cpp  

- IPCThreadState.cpp  

- ProcessState.cpp  

- IServiceManager.cpp  

- IInterface.cpp  

- Parcel.cpp 

#### frameworks/native/include/binder/ 

- IInterface.h (包括BnInterface, BpInterface)
- Binder.h 也就是BBinder

https://blog.csdn.net/shift_wwx/article/details/100104542

https://blog.csdn.net/zbunix/article/details/8758631

​    BpBinder是client端创建的用于消息发送的代理，而BBinder是server端用于接收消息的通道。