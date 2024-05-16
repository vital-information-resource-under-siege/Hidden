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

#### Plan B:-

I came to the conclusion that D-Bus is implemented pretty well and finding vulnerabilities in the D-Bus itself is hard.

After some hours of brainstorming session, we next came to the plan of creating our own application that sends IPC messages to the root binary to perform the privileged actions.

So I spent some hours on reading the [Glib Documentation](https://docs.gtk.org/glib/) and wrote a C Code with multiple errors and finally after a lot of tries the code was working.

```c
#include <stdio.h>
#include <gio/gio.h>

int main() {
    GError *error = NULL;
    GDBusConnection *connection;
    GDBusMessage *message, *reply;
    GVariant *variant;
    GDBusProxy *proxy;
    int ret;
    // Synchronously get the session bus
    connection = g_bus_get_sync(1, NULL, &error);
    if (error != NULL) {
        g_printerr("Error connecting to bus: %s\n", error->message);
        g_error_free(error);
        return 1;
    }
    proxy = g_dbus_proxy_new_sync(connection,0,0,"dbus_service_name","/object_path","interface_name",0,&error);
    variant = g_variant_new("data_type","data");
    variant = g_dbus_proxy_call_sync(proxy,"method_name",variant,0,60000,0,&error);
    g_variant_get(variant,"return_value_data_type",&ret);
    printf("The server answered with %d\n",ret);
}
```

The code was working but the root application was not performing the required action,it was rejecting the IPC message we are sending.

After all those reading I have done from the D-Bus Documentation, I am pretty sure that this wasn't the behaviour of D-Bus.

My attention turned towards the application binary once again and scroll through some decompiled code again. Then there was this function call with the name 

```c++
pcVar16 = (char *)dbus_message_get_sender(param_2);
cVar8 = secret_object_name::verifyRPCClient(this,pcVar16);
```

This verifyRPCClient function inturn calls verifySignature which calls signedelf function with a RSAPublicKey to verify the binary which has been sending the IPC messages.  

![meme 4](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/plan.jpg)



![meme 5](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/2_vs_1.jpg)

