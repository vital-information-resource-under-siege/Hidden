# Breaking Down Barriers: Exploiting Authenticated IPC Clients

**Inter-Process Communication (IPC)** is a critical aspect of modern computing, enabling different processes within an operating system to exchange data and coordinate actions. Through various methods like message passing, shared memory, and sockets, IPC facilitates seamless interaction between distinct applications, enhancing system efficiency and functionality. However, the security of these communications is paramount, as vulnerabilities in authenticated IPC clients can lead to severe breaches and exploitation. 

## Story on how we end up to this blog

#### Recon :-

So me and my team had an application  in hand that consists of two binaries one with normal user privileges and the other one with root privileges.

So, our first plan in our mind is to attack the IPC component itself. So we started the recon process to know about the IPC mechanism on how the low privileges application passes messages to the root privileges application.

The initial thoughts before reversing the application we thought the IPC component might be a Shared Memory, Message Queues, Sockets,Pipes or Signals. If the implementation is any of this, we already had an idea on how to attack these implementation.

After a short time of reversing and going through the decompilation code we found the application was using **D-Bus** to send and receive IPC messages.

![meme 1](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus_function.png)



#### D-BUS :-

D-Bus (Desktop Bus) is an inter-process communication (IPC) system widely used in Unix-like operating systems. It allows multiple software applications to communicate with one another in a standardized way, facilitating coordination and data sharing. D-Bus supports both system-wide and user-session communication, making it integral to modern desktop environments and system services.

So, we spent some time on learning about D-Bus by visiting its [documentation](https://www.freedesktop.org/wiki/Software/dbus) and then we came across this cool [blog](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/d-bus-enumeration-and-command-injection-privilege-escalation) on how to enumerate and perform privilege escalation on D-Bus.

![meme 2](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus.jpg)











So this was an idea thought about me and my teammates where our end goal was to make unauthenticated calls to a specific IPC client to perform privileged actions.

![meme 3](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/plan.jpg)