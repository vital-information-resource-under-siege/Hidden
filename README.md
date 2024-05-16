# Breaking Down Barriers: Exploiting Authenticated IPC Clients

![start](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/start.jpg)

**Inter-Process Communication (IPC)** is a critical aspect of modern computing, enabling different processes within an operating system to exchange data and coordinate actions. Through various methods like message passing, shared memory, and sockets, IPC facilitates seamless interaction between distinct applications, enhancing system efficiency and functionality. However, the security of these communications is paramount, as vulnerabilities in authenticated IPC clients can lead to severe breaches and exploitation. 

## Story on how we end up to this blog

#### Recon :-

So,Me and my team had an application  in hand that consists of two binaries one with normal user privileges and the other one with root privileges.

The application has some set of credentials that checks upon it on the server side and then sends an IPC message from the binary with normal user privileges to the binary with root privileges to perform some **privileged actions** .

The end goal was to perform the privileged actions without providing any credentials to the application. 

Our first plan in our mind is to attack the IPC component itself. So we started the recon process to know about the IPC mechanism on how the low privileges application passes messages to the root privileges application.

The initial thoughts before reversing the application we thought the IPC component might be a Shared Memory, Message Queues, Sockets,Pipes or Signals. If the implementation is any of this, we already had an idea on how to attack these implementation.

After a short time of reversing and going through the decompilation code we found the application was using **D-Bus** to send and receive IPC messages.

The application uses GLib API's to use the D-Bus to send and receive IPC messages .

![meme 1](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus_function.png)

#### D-BUS :-

D-Bus (Desktop Bus) is an inter-process communication (IPC) system widely used in Unix-like operating systems. It allows multiple software applications to communicate with one another in a standardized way, facilitating coordination and data sharing. D-Bus supports both system-wide and user-session communication, making it integral to modern desktop environments and system services.

So, we spent some time on learning about D-Bus by visiting its [documentation page](https://www.freedesktop.org/wiki/Software/dbus) and also we came across this [cool blog](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/d-bus-enumeration-and-command-injection-privilege-escalation) on hacktricks on how to enumerate and perform privilege escalation on D-Bus.

![meme 2](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus.jpg)

After a long time spent on learning about D-Bus and performing some enumeration as described by the Blog.We finally came to this conclusion that attacking the D-Bus is super complicated and it is not as easy as attacking the custom implementation like  I thought it to be.

![meme 3](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus_vs_me.jpg)



I started creating a binary that tries to sends the same IPC messages to perform the privileged actions. 

![meme ](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/plan.jpg)