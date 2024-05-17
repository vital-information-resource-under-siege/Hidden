# Breaking Down Barriers: Exploiting Authenticated IPC Clients

![start](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/start.jpg)

**Inter-Process Communication (IPC)** is a critical aspect of modern computing, enabling different processes within an operating system to exchange data and coordinate actions. Through various methods like message passing, shared memory, and sockets, IPC facilitates seamless interaction between distinct applications, enhancing system efficiency and functionality. However, the security of these communications is paramount, as vulnerabilities in authenticated IPC clients can lead to severe breaches and exploitation. 

## Story on how we end up on this blog

#### Recon :-

So, Our team was assessing an application that consisted of two binaries one with normal user privileges and the other one with root privileges.

The application has some set of credentials that checks upon it on the server side and then sends an IPC message from the binary with normal user privileges to the binary with root privileges to perform some **privileged actions** .

The end goal was to perform the privileged actions without providing any credentials to the application. 

Our first plan in our mind is to attack the IPC component itself. So we started the recon process to know about the IPC mechanism on how the low privileges application passes messages to the root privileges application.

The initial thoughts before reversing the application we thought the IPC component might be a Shared Memory, Message Queues, Sockets, Pipes or Signals. If the implementation is any of these, we already have an idea of how to attack this implementation.

After a short time of reversing and going through the decompilation code, we found the application was using **D-Bus** to send and receive IPC messages.

The application uses GLib API's to use the D-Bus to send and receive IPC messages.

![meme 1](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus_function.png)

#### D-BUS :-

D-Bus (Desktop Bus) is an inter-process communication (IPC) system widely used in Unix-like operating systems. It allows multiple software applications to communicate with one another in a standardized way, facilitating coordination and data sharing. D-Bus supports both system-wide and user-session communication, making it integral to modern desktop environments and system services.

So, we spent some time learning about D-Bus by visiting its [documentation page](https://www.freedesktop.org/wiki/Software/dbus) and also we came across this [cool blog](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/d-bus-enumeration-and-command-injection-privilege-escalation) on hacktricks on how to enumerate and perform privilege escalation on D-Bus.

![meme 2](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus.jpg)

After a long time spent learning about D-Bus and performing some enumeration as described by the Blog.We finally came to the conclusion that attacking the D-Bus is super complicated and it is not as easy as attacking the custom implementation like  I thought it to be.

![meme 3](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/dbus_vs_me.jpg)

#### Plan B:-

We came to the conclusion that D-Bus is implemented pretty well and finding vulnerabilities in the D-Bus itself is hard.

After some hours of brainstorming session, we next came to the plan of creating our application that sends IPC messages to the root binary to perform the privileged actions.

So we spent some hours on reading the [Glib Documentation](https://docs.gtk.org/glib/) and wrote a C Code with multiple errors and finally after a lot of tries the code was working.

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

The code was working but the root application was not performing the required action, it was rejecting the IPC message we were sending.

After all the reading I have done from the D-Bus Documentation, I am pretty sure that this wasn't the behaviour of D-Bus.

Our attention turned towards the application binary once again and scroll through some decompiled code again. Then there was this function call with the name 

```c++
pcVar16 = (char *)dbus_message_get_sender(param_2);
cVar8 = secret_object_name::verifyRPCClient(this,pcVar16);
```

This verifyRPCClient function in turn calls verifySignature which calls signedelf function with a RSAPublicKey to verify the binary which has been sending the IPC messages.  

![meme 4](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/plan.jpg)

We were going through the busctl monitor as well, Whenever I was trying to pass the message the D-Bus showed an error to confirm the authentication was taking place.

In my mind, I was still thinking of a way that would allow me to perform unauthenticated IPC calls. Finally, reality hit me that unauthenticated IPC calls can't be made to this application. 

**Plan B** also failed miserably.

#### Plan C:-

Both the plans have failed miserably because of the secure implementation of the application as well as the D-bus.

![meme 5](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/2_vs_1.jpg)

The only way is to send authenticated IPC messages without the need for credentials. However, the issue is that the messages are only sent when the credentials are provided. 

We explored all these previous options we had, Every issue we found in the application that we can use to maybe aid us here in this goal.

We already identified an issue of **Shared Objection Injection** that is present in the application that runs on lower privileges. However, the issue at first seemed to be without any impact as the goal was to perform privileged actions.

#### Shared Object Injection:-

Shared object injection on Linux is a technique used to load a shared library (shared object, `.so` file) into the address space of a running process. This allows the injected code to execute within the context of the target process, giving it access to the process’s memory, resources, and execution flow. 

![meme 5](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/osaka.jpg)

The Shared Object Injection vulnerability came into play again where we can use it to intercept the library calls we make by passing it through our own shared library.

We searched for the attack surface, finding multiple function points that could alter the code flow to reach our goal.

After we spent hours on finding the attack surface, for performing the attack in a stable way.

Our team found something very serious the user privileges application was sending critical sensitive data to the root privileges application through IPC which can be smuggled using our injection bug that can have more impact than the former goal we were pursuing. 

![meme 6](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/after_before.jpg)

This single bug can now trigger two critical points of the application that can have the maximum impact.

1)Without providing any credentials, we can bypass the application logic flow and perform the actions it wasn't supposed to.

2)The application with normal user privileges sent sensitive critical data to the application with root privileges through D-Bus.

By intercepting the library calls to the application and injection our own code, we were able to perform the privileged action without any authentication and also get some sensitive data leak.

![meme 7](https://github.com/vital-information-resource-under-siege/Hidden/blob/main/Images/impact.jpg)





