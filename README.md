# Breaking Down Barriers: Exploiting Authenticated IPC Clients

**Inter-Process Communication (IPC)** is a critical aspect of modern computing, enabling different processes within an operating system to exchange data and coordinate actions. Through various methods like message passing, shared memory, and sockets, IPC facilitates seamless interaction between distinct applications, enhancing system efficiency and functionality. However, the security of these communications is paramount, as vulnerabilities in authenticated IPC clients can lead to severe breaches and exploitation. 

So this was an idea thought about me and my teammates where our end goal was to make unauthenticated calls to a specific IPC client to perform privileged actions.

![meme 1](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/plan.jpg)

So me and my team had an application  in hand that consists of two binaries one with normal user privileges and the other one with root privileges.

So, our first plan in our mind is to attack the IPC component itself. So we started the recon process to know about the IPC mechanism on how the low privileges application 

![meme 2](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus.jpg)

