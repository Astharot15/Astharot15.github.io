+++
date = '2026-02-18T15:42:02Z'
draft = false
title = 'Kernel Level Keylogger'
description = 'A guide on creating a kernel level keylogger for CTF challenges'
+++

```
Disclaimer: I am not a native English speaker and I do not like to use AI, so everything is hand written by me.
Disclaimer: There is not any code from my keylogger, for now it will keep private and everything explained in the post is the theoretical approach to kernel level keyloggers.
```

In this post I am talking about how keystrokes work and how I used it to create a kernel level keylogger with a very simple technique, a filter driver. I created this driver for the [HackOn](https://hackon.es) CTF (my university CTF) as a reversing challenge where players had to discover how it works and type the flag to deactivate the keylogger, creating a scenario where an attacker had a way to avoid self-infection.

# Index

> [Keystroke Adventures](#keystroke-adventures) \
> [Filter Driver](#filter-driver) \
> [Real World Considerations](#real-world-considerations) \
> [Next steps](#next-steps)

# Keystroke Adventures

During my research I found a `Emre TINAZTEPE` training called [The Adventures of a Keystroke](https://opensecuritytraining.info/Keylogging.html). It explains the path a keystroke takes to convert keyboard input to OS level. 

To simplify everything, we can describe the route like this:

1. PS/2 controller translates keystroke from the keyboard into scan codes.
2. i8042ptr.sys driver handles interrupts and passes them to kbdclass.sys.
3. kbdclass.sys reads the data with a read routine and process the keystroke.

There are a lot of steps between and after those but this is enough to understand and create this driver.

Another important part is the communication between i8042ptr.sys and kbdclass.sys, it is necessary to attach Device stacks, which we are using later.

# Filter Driver

Device Objects are instances of the DEVICE_OBJECT structure and they are associated with drivers. Each Driver has a "Driver Stack". The first Device Object created in the Device Stack is in the bottom position and last one is at the top. According to the Microsoft Documentation:

> `Filter drivers play auxiliary roles in processing read, write, and device control requests. As each function and filter driver is loaded, it creates a device object and attaches itself to the device stack.`

[Filter Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/device-nodes-and-device-stacks)

That means that it is possible to create a new Device Object and attach to Device Stack and everything going through the stack is passes through our Driver Read routine. So, if we modify the Read Routine we can implement any malicious actions.

In this case, the keylogger was created for a CTF challenge so this Read Routine is only checking the flag.

To make this I used two functions `IoCreateDevice` and `IoGetDeviceObjectPointer`. First one creates the Device Object that will be Lower Device and the second one will attach Source Device to Target Device.

Our Filter Driver should have some Major Functions to work properly. First one is `IRP_MJ_CREATE` where is defined what the driver will do when is loaded. Then `IRP_MJ_CLOSE` that is mandatory when a driver has an exception and need to be removed and finally `IRP_MJ_READ` where we are implementing some functionallity to read keyboard input. I did not implement a cleanup function, this routine is triggered when the driver is unloaded and this is supposed to be a persistent keylogger, so it is not the objective.

This is how device stack looks before driver is loaded:

![](/images/device_stack.png)

And here is after the new driver is loaded:

![](/images/device_stack_keylogger.png)

Then, in order to access keystroke data, it is necessary to extract from IRP(I/O request packets). Those packets contain the information through device stack. Dispatch Read, which is the READ Major Function in Driver Object, has this prototype:

```
NTSTATUS DriverDispatch(
  [in, out] _DEVICE_OBJECT *DeviceObject,
  [in, out] _IRP *Irp
)
```

Keystrokes are not only codes, there is a structure called [_KEYBOARD_INPUT_DATA](https://learn.microsoft.com/en-us/windows/win32/api/ntddkbd/ns-ntddkbd-keyboard_input_data).

```
typedef struct _KEYBOARD_INPUT_DATA {
  USHORT UnitId;
  USHORT MakeCode;
  USHORT Flags;
  USHORT Reserved;
  ULONG  ExtraInformation;
} KEYBOARD_INPUT_DATA, *PKEYBOARD_INPUT_DATA;
```

Once we know that keystroke codes are in Irp->AssociatedIrp.SystemBuffer->MakeCode it is possible to log every single keystroke and create a communication with the C2.

# Real World Considerations

Now the question is, so is it that easy to load a kernel level keylogger in a real world machine? While it is technically possible, I mean, I did it in my machine, but there are some considerations that will make you think twice before doing it in a corporate endpoint.

First, to load a driver you need to be local admin in the machine.
Second, there is a little thing called DSE(Driver Signature Enforcement) which we can ignore, but it is not that simple.
Third, EDRs also have some kernel callbacks so it is possible to be detected.

Lets explain a little bit more DSE. This protection makes Windows allow to load only signed Drivers, you can tell your computer "Hey, I am a developer and I want to test this unsigned driver" and enable test signing mode which will self sign a Driver with a "test" certificate.

Now we are going into unexplored territory for me, does the EDR track those untrusted drivers loading? I did not test it but checking some [Elastic Rules](https://elastic.github.io/detection-rules-explorer/rules/d8ab1ec1-feeb-48b9-89e7-c12e189448aa) it is very probable that at least, EDR will be heavily focus on your driver.

# Next steps

With those consideration I can stop here and go do other stuff, but I enjoyed a lot driver development so I have some things to do.

DSE problem can actually be bypassed. If you have a R/W kernel primitive, it is possible to execute unsigned drivers. There are some tools that do this but I would like to implement a exploit of some CVE to execute my own keylogger.

Then, this is only a CTF challenge, it does not actually work as a keylogger. There are two future steps, first implement any way to exfiltrate keystrokes with address several cahllenges, where to store, where to send, how it can be sent,... And then implement communication with a C2, it will probably be Adaptix or Sliver, but I am open to advice.

Finally change all the keylogger, it is really cool but I would like to implement other technique more stealthy, this was the easiest way, but there are more techniques where I need to bypass some kernel protections.

Thank you for reading, this is my first blog but not last, I hope you enjoyed it.

I am open to receive tips to improve everything, if you have any suggestion contact me on linkedin or X.