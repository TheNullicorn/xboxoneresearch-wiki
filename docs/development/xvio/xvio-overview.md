# Overview of XVIO
The *XVIO driver* (`xvio.sys`) facilitates all communication between the virtualized partitions. It's APIs share some similarity with the [Hyper-V Inter-Partition Communication](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/main/tlfs/Hypervisor%20Top%20Level%20Functional%20Specification%20v6.0b.pdf){:target='blank'} APIs. If you're interested in more details, this document should still apply to XVIO, atleast some-what.

Whilst each partition has it's own version of `xvio.sys`, the differences appear to be minor, as each driver is simply being built using different preprocessor definitions to target different partitions.  

## Service identifiers  
The service identifier is an ID that is unique to an instance of XVIO that is initialized using *[XvioInitialize](./xvio-initialize.md).* In total there are 32 unique identifiers available to a partition. This number is finite as all the services are stored within an array in `xvio.sys`'s data section, where services are allocated to drivers where needed.  

You may find it helpful to think of a service ID as a driver ID. This is because when communicating between partitions, it is the service ID that specifies what driver (on the remote partition) to communicate with. For example, `xvmm.sys` *- the virtual machine manager on HostOS -* uses the service identifier `0xf`. Thus, to communicate with this driver, `xvmctrl.sys` *- the virtual machine control driver on SystemOS -* uses that same ID to send requests to the VM manager. Furthermore, for the sake of continuity, it is common for drivers with similar functionality to use the same service ID on different partitions, despite it not being strictly necessary from a functional stand-point. With this in mind, we can be certain that all drivers related to virtual machine management and control allocate the service ID `0xf` on their respective operating systems.

### List of service identifiers

> **Note**<br>
> Information below is based on firmware version **10.0.25398.4909**. Services can and have been added, removed, and
> reassigned IDs.

| ID     | Internal Name[^internal-names] | HostOS Driver       | SystemOS Driver                    | GameOS Driver |
|--------|--------------------------------|---------------------|------------------------------------|---------------|
| `0x00` | `INTERNAL`                     | `Xvio.sys`          | `Xvio.sys`                         | ?             |
| `0x01` | `GRAPHICS`                     | `HostKmd.sys`       | `SraKmd.sys`<br>`SraKmd_Arden.sys` | ?             |
| `0x02` | `GRAPHICS_EX`[^graphics-ex]    | ?                   | ?                                  | ?             |
| `0x03` | `STORAGE`                      | `XvsP.sys`          | `XvsC.sys`                         | ?             |
| `0x04` | `NETWORK`                      | `XvnP.sys`          | `XvnC.sys`                         | ?             |
| `0x05` | `INPUT`                        | `InputXVIOHost.sys` | `InputXVIOClient.sys`              | ?             |
| `0x06` | `ODD`                          | `XvsP.sys`          | `XvsC.sys`                         | ?             |
| `0x07` | `AUDIO`                        | ?                   | `VirtualAudioBus.sys`              | ?             |
| `0x08` | `PIPE`                         | `Vmnp.sys`          | `Vmnp.sys`                         | ?             |
| `0x09` | `KEYBOARD`                     | ?                   | `XvioKbd.sys`                      | ?             |
| `0x0a` | `IRBLASTER`                    | ?                   | `IrBlasterSra.sys`                 | ?             |
| `0x0b` | `NUISENSOR`                    | `PetraXP.sys`       | `PetraXC.sys`                      | ?             |
| `0x0c` | `NETWORK_WIFI`                 | `XvnP.sys`          | `XvnC.sys`                         | ?             |
| `0x0d` | `NETWORK_WIFI_DIRECT`          | `XvnP.sys`          | `XvnC.sys`                         | ?             |
| `0x0e` | `UPDATE`                       | `XvuP.sys`          | `XvuC.sys`                         | ?             |
| `0x0f` | `VM_MGMT`                      | `Xvmm.sys`          | `XVmCtrl.sys`                      | ?             |
| `0x10` | `PSP`                          | `Psp.sys`           | `PspSra.sys`                       | ?             |
| `0x11` | `NETWORK_WFD_ROLE0`            | `XvnP.sys`          | `XvnC.sys`                         | ?             |
| `0x12` | `NETWORK_WFD_ROLE1`            | `XvnP.sys`          | `XvnC.sys`                         | ?             |
| `0x13` | `LIVETV`                       | ?                   | `HdmiInC.sys`                      | ?             |
| `0x14` | `XVIOMON`                      | `XvioMonP.sys`      | `XvioMonC.sys`                     | ?             |
| `0x15` | `ZURICH`                       | ?                   | `Zurich.sys`                       | ?             |
| `0x16` | `GRAPHICS_EX`[^graphics-ex]    | ?                   | `SraExKmd.sys`                     | ?             |
| `0x17` | `XTC`                          | ?                   | `XtcC.sys`                         | ?             |
| `0x18` | `USB`                          | ?                   | `Xvuh.sys`                         | ?             |
| `0x19` | `XTF`                          | ?                   | `XbtpLinkC.sys`                    | ?             |
| `0x1a` | `MOUSE`                        | ?                   | `XvioMou.sys`                      | ?             |
| `0x1b` | `XVSOCKET`                     | `Xvmm.sys`          | `XvBus.sys`                        | ?             |
| `0x1c` | `NETWORK_GS`                   | `XvnP.sys`          | `XvnC.sys`                         | ?             |
| `0x1d` | `RESERVED0`                    | ?                   | ?                                  | ?             |
| `0x1e` | `TEST`                         | ?                   | ?                                  | ?             |

[^internal-names]: Internal names are found inside the HostOS utility `C:\Windows\System32\xvtool.sys`

[^graphics-ex]: GRAPHICS_EX was moved from ID 0x02 to 0x16 at some point, however, the internal name is still mapped to
both. It is unknown if or what 0x02 is currently used for

## Partition identifiers
A lot of the XVIO functions take a partition ID as an argument, allowing for IO with a specified partition. An enumeration outlining these IDs can be found below:  
```cpp
enum PARTITION_ID {
    Any      = 0,
    HostOS   = 1,
    SystemOS = 2,
    GameOS   = 3
};
```

## Very, very important disclaimer
This research is not complete! There's plenty more functions within the XVIO drivers that I simply haven't gotten around to analysing or documenting. For reference, there's around 67 exported functions (excluding DllInitialize and DllUnload) within `xviosra.sys`.  
Oh and also, a lot of information may be missing or some information may seem abstract as we currently cannot know how the hypervisor is actually handling XVIO-related hypercalls. That being said I aim to add a bunch more info on how HostOS is handling things at a lower level! :)