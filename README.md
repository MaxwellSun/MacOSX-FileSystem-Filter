# MacOSX-FileSystem-Filter
A file system filter for Mac OS X

##License

The license model is a BSD Open Source License. This is a non-viral license, only asking that if you use it, you acknowledge the authors, in this case Slava Imameev.

The project uses the distorm disassembler https://github.com/gdabah/distorm which is now released under BSD license.

##Design
Mac OS X doesn't support a full fledged file system filtering like Windows as Apple believes that BSD stackable file system doesn't fit Mac OS X. The available kernel authorization subsystem ( kauth ) allows only filtering open requests and a limited number of operations on file/directory. It doesn't allow filtering read and write operations and provides a limited control over file system operations as a vnode is already created at the moment of kauth callback invoking. In Windows parlance a kauth callback is a postoperation callback for create/open request ( IRP_MJ_CREATE for Windows ), that is all you have for Mac OS X. Not much really.

 The filtering can also be implemented by registering MAC ( Mandatory Access Control ) layer. But MAC has limited functionality and not consistent in relation to file system filtering as it was not designed as a file system filter layer. Instead of being called by VFS layer MAC registered callbacks are scattered through the kernel code. They are called from system calls and other kernel subsystems, so MAC doesn't provide a consistent interface for VFS filter and if I remember correctly MAC was declared as deprecated for usage by third party developers.

  To summarize the above, there is no official support from Apple if you need to filter read or/and write requests, filter and modify VFS vnode operation.

The lack of a stackable file system support by Mac OS X VFS required to find a way to place a filter between VFS invoking vnode operations via VNOP_* functions and a file system driver implementing these vnode operations. I developed a hooking technique for VFS layer that emulates a stackable file system by replacing vnode operations in array. This technique allows to place a filter between VFS and file system driver and supports sophisticated filtering such as isolation file system filter when a filter creates vnodes instead of a file system driver thus gaining a full control over file data. I used this technique in two projects to implement filtering for lookup, create, read and write requests and to implement an isolation file system.

 FltFakeFSD.h and FltFakeFSD.cpp are optional files that helps to infer the vnode and vnodeop_desc structures layout that are not declared in SDK.  This is achieved by registering a dummy file system, creating a vnode and inspecting it to find required offsets. All you need to do is to call FltGetVnodeLayout() in a filter initialization code.

 Alternatively you can extract declarations from XNU code at opensource.apple.com. An alternative implementation without using the fake FSD can be found in VersionDependent.h and VersionDependent.cpp files, where struct vnodeop_desc_Yosemite was borrowed from Apple's open source code for Yosemite(10.10), it happened that vnode and vnodeop_desc structures haven't change in all latest kernel versions so this code works for Mavericks(10.9) and El Capitan(10.11). With Sierra(10.12) release the field offsets changed for vnode structure and the project was changed to be compiled with FltFakeFSD.cpp to infer vnode layout. If you want to switch back then remove USE_FAKE_FSD preprocessor definition from project settings and replace vnode_Yosemite structure definition.

 The filter registers a kauth callback FltIOKitKAuthVnodeGate::VnodeAuthorizeCallback which in turn calls FltHookVnodeVopAndParent that hooks vnode related operations. The kauth callback is used only to trigger filtering for a file system. When an application opens a file FileX in a directory DirX a vnode for a DirX is provided as a parameter for VnodeAuthorizeCallback so the following calls to file system's lookup or create operations for FileX will be visible for file system filter. When an original lookup or create returns a filter calls FltHookVnodeVopAndParent for a returned vnode or does this in a kauth callback which is nearly the same as kauth callback is a postoperation callback called after VNOP_LOOKUP or VNOP_CREATE. In case of lookup vnode operation do not forget about name cache, so lookup is not always called when an application calls open( FileX ), but if you need to see all lookup operations it is possible to disable name caching for a vnode. You can devise your own way of finding vnodes to hook depending on your requirements, for my purposes a kauth callback worked well as I was filtering vnodes of VREG type so I can use VDIR vnode as an anchor that started filtering for VREG vnodes. Also in most file systems vnode operations array is shared between all vnodes of VREG and VDIR type.

 The filter registers operations it wants to intercept in gFltVnodeVopHookEntries array. The skeleton filter immediately calls an original function from a hook but if you want to see a filter in action look at DldVNodeHook.cpp ( https://github.com/slavaim/MacOSX-Kernel-Filter/blob/master/DLDriver/DldVNodeHook.cpp ) which is a more advanced implementation at MacOSX-Kernel-Filter that also implements an isolation filter for read and write operations on a file by redirecting read and write to a sparse file on another volume. The isolation filter is implemented at DldCoveringVnode.cpp and a sparse file at DldSparseFile.cpp
 
##Application
You can find a real world filter application in MacOSX-VFS-redirector( https://github.com/slavaim/MacOSX-VFS-redirector ) and a file system isolation filter( https://github.com/slavaim/MacOSX-Kernel-Filter/blob/master/DLDriver/DldVNodeHook.cpp , it is a part of a bigger project).

##Example of call stacks when a filter is loaded
FYI a set of call stacks when hooks are active  

Lookup hook  
 ```
 frame #0: 0xffffff7f903b7a11 FsdFilter``FltVnopLookupHook(ap=0xffffff80a7053a18) + 17 at VFSHooks.cpp:65  
 frame #1: 0xffffff800dd3f2f8 kernel``lookup(ndp=0xffffff80a7053d58) + 968 at kpi_vfs.c:2783  
 frame #2: 0xffffff800dd3ea95 kernel``namei(ndp=0xffffff80a7053d58) + 1941 at vfs_lookup.c:371  
 frame #3: 0xffffff800dd52005 kernel``nameiat(ndp=0xffffff80a7053d58, dirfd=<unavailable>) + 133 at vfs_syscalls.c:2920  
 frame #4: 0xffffff800dd5fcc7 kernel``fstatat_internal(segflg=<unavailable>, ctx=<unavailable>, path=<unavailable>, ub=<unavailable>, xsecurity=<unavailable>, xsecurity_size=<unavailable>, isstat64=<unavailable>, fd=<unavailable>, flag=<unavailable>) + 231 at vfs_syscalls.c:5268  
 frame #5: 0xffffff800dd54f0a kernel``stat64(p=<unavailable>, uap=0xffffff801e3f1380, retval=<unavailable>) + 58 at vfs_syscalls.c:5413  
 frame #6: 0xffffff800e04dcb2 kernel``unix_syscall64(state=0xffffff801dfc5540) + 610 at systemcalls.c:366  
 ```
 
 Pagein hook  
 ```
 frame #0: 0xffffff7f903b7b91 FsdFilter``FltVnopPageinHook(ap=0xffffff80a76aba20) + 17 at VFSHooks.cpp:229  
 frame #1: 0xffffff800e046372 kernel``vnode_pagein(vp=0xffffff801904db40, upl=0x0000000000000000, upl_offset=<unavailable>, f_offset=73416704, size=<unavailable>, flags=0, errorp=0xffffff80a76abae8) + 402 at kpi_vfs.c:4980  
 frame #2: 0xffffff800db952a8 kernel``vnode_pager_cluster_read(vnode_object=0xffffff80a76abb10, base_offset=73416704, offset=<unavailable>, io_streaming=<unavailable>, cnt=<unavailable>) + 72 at bsd_vm.c:1045  
 frame #3: 0xffffff800db943f3 kernel``vnode_pager_data_request(mem_obj=0xffffff8018fb5e38, offset=73433088, length=<unavailable>, desired_access=<unavailable>, fault_info=<unavailable>) + 99 at bsd_vm.c:826  
 frame #4: 0xffffff800db9f75b kernel``vm_fault_page(first_object=0x0000000000000000, first_offset=73433088, fault_type=1, must_be_resident=0, caller_lookup=0, protection=0xffffff80a76abe84, result_page=<unavailable>, top_page=<unavailable>, type_of_fault=<unavailable>, error_code=<unavailable>, no_zero_fill=<unavailable>, data_supply=0, fault_info=0xffffff80a76abbe0) + 3051 at memory_object.c:2178  
 frame #5: 0xffffff800dba3902 kernel``vm_fault_internal(map=0xffffff8010f761e0, vaddr=140735436247040, fault_type=1, change_wiring=0, interruptible=2, caller_pmap=0x0000000000000000, caller_pmap_addr=<unavailable>, physpage_p=0x0000000000000000) + 3042 at vm_fault.c:4423  
 frame #6: 0xffffff800dc1ec9c kernel``user_trap(saved_state=<unavailable>) + 732 at vm_fault.c:3229  
 ```
 
 Close hook  
```
frame #0: 0xffffff7f903b7a91 FsdFilter``FltVnopCloseHook(ap=0xffffff80a78cbd28) + 17 at VFSHooks.cpp:119  
frame #1: 0xffffff800dd734b7 kernel``VNOP_CLOSE(vp=0xffffff801c7144b0, fflag=<unavailable>, ctx=<unavailable>) + 215 at kpi_vfs.c:3047  
frame #2: 0xffffff800dd671d1 kernel``vn_close(vp=0xffffff801c7144b0, flags=<unavailable>, ctx=0xffffff80a78cbe60) + 225 at vfs_vnops.c:723  
frame #3: 0xffffff800dd6850f kernel``vn_closefile(fg=<unavailable>, ctx=0xffffff80a78cbe60) + 159 at vfs_vnops.c:1494  
frame #4: 0xffffff800dfb1680 kernel``closef_locked [inlined] fo_close(fg=0xffffff801ba367e0, ctx=0xffffff801bcb42a0) + 14 at kern_descrip.c:5711  
frame #5: 0xffffff800dfb1672 kernel``closef_locked(fp=<unavailable>, fg=0xffffff801ba367e0, p=0xffffff8019600650) + 354 at kern_descrip.c:4982  
frame #6: 0xffffff800dfad88e kernel``close_internal_locked(p=0xffffff8019600650, fd=<unavailable>, fp=0xffffff801fc113d8, flags=<unavailable>) + 542 at kern_descrip.c:2765  
frame #7: 0xffffff800dfb13d6 kernel``close_nocancel(p=0xffffff8019600650, uap=<unavailable>, retval=<unavailable>) + 342 at kern_descrip.c:2666  
frame #8: 0xffffff800e04dcb2 kernel``unix_syscall64(state=0xffffff801bc20b20) + 610 at systemcalls.c:366  
```

Inactive hook  
```
frame #0: 0xffffff7fa9dc6ac0 FsdFilter``FltVnopInactiveHook(ap=0xffffff803c5198cc) at VFSHooks.cpp:140  
frame #1: 0xffffff8027775e06 kernel``VNOP_INACTIVE(vp=0xffffff803c519870, ctx=<unavailable>) + 70 at kpi_vfs.c:4756  
frame #2: 0xffffff8027745543 kernel``vnode_rele_internal(vp=0xffffff80bfecbd90, fmode=<unavailable>, dont_reenter=<unavailable>, locked=<unavailable>) + 435 at vfs_subr.c:1929  
frame #3: 0xffffff80279c5359 kernel``proc_exit(p=0xffffff80338eb000) + 2841 at vfs_subr.c:1845  
frame #4: 0xffffff802756447a kernel``thread_terminate_self + 474 at thread.c:466  
frame #5: 0xffffff802756878b kernel``special_handler(rh=<unavailable>, thread=<unavailable>) + 219 at thread_act.c:891  
frame #6: 0xffffff802756855a kernel``act_execute_returnhandlers + 202 at thread_act.c:801  
frame #7: 0xffffff8027536e76 kernel``ast_taken(reasons=<unavailable>, enable=<unavailable>) + 278 at ast.c:177  
frame #8: 0xffffff802761eeae kernel``i386_astintr(preemption=<unavailable>) + 46 at trap.c:1171  
```

##Some words on unloading implementation
The module has been made non-unloadable by taking a reference to IOKit com\_FsdFilter class in com\_FsdFilter::start routine. The reason is that unload for a hooking file system filter driver is a dangerous operation as it is hard to implement it correctly to avoid race conditions and preserve data consistency. To process unload correctly you need to process the following cases  
  
- unhook all vnode tables  
- check that there is no return to the hooking code in any call stack or else unloading a module results in a panic as control returns to unloaded code  
- check that there is no hooking code being called, i.e. there is no a processor with IP register pointing to any function in a driver  
- ensure that data changed by a filter will not crash the system or damage user or system data, for example if the driver implements any data modification it should guarantee data consistency on unload  
  
The only feasible way to implement this is dividing a driver in two parts - one part can be unloaded and another part is never unloaded. Unloading complicates the code as you have to synchronize the code with unloading procedure, this increases complexity and as a result of this complexity and nondeterminism in the system state it's never implemented correctly
