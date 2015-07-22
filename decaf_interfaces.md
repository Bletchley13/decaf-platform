# Interfaces #

> DECAF provides many event-driven interfaces to make instrumentation easier. There are two kinds of interfaces.The first is callback for you to insert instrumentation code at specific event. Another set of interfaces is used as utils to get information of guest os or read memory from guest OS.

## 1 Callback interfaces ##

> To get a better understanding of these interfaces, a basic understanding of QEMU translation part is needed,please see [qemu](http://wiki.qemu.org/Manual) for more details.

### 1.1 Callback types ###

  1. **VMI callback**
> it’s defined in /shared/vmi\_callback.h:VMI\_callback\_type\_t
> VMI\_CREATEPROC\_CB:  invoked when eprocess structure is created

> VMI\_REMOVEPROC\_CB:  invoked when delete the process

> VMI\_LOADMODULE\_CB: invoked when dll or driver is loaded

> VMI\_LOADMAINMODULE\_CB: invoked when process starts to run

> VMI\_PROCESSBEGIN\_CB: alias of VMI\_LOADMAINMODULE\_CB



> 2. **Instruction callback**

> it’s defined in  /shared/DECAF\_callback\_common.h:DECAF\_callback\_type\_t
This callback instruments guest operating system  at instruction level.

> DECAF\_BLOCK\_BEGIN\_CB : invoked at the start of the block.

> DECAF\_BLOCK\_END\_CB: invoked at the end of the block.

> DECAF\_INSN\_BEGIN\_CB: invoked before this instruction is executed

> DECAF\_INSN\_END\_CB:invoked after this instruction is executed


> 3. **Mem read/write callback**
it’s defined in  /shared/DECAF\_callback\_common.h:DECAF\_callback\_type\_t

DECAF\_MEM\_READ\_CB :invoked when the memory read operation has been done

DECAF\_MEM\_WRITE\_CB:invoked when the memory write operation has been done

> 4. **Keystroke callback**

it’s defined in  /shared/DECAF\_callback\_common.h:DECAF\_callback\_type\_t

DECAF\_KEYSTROKE\_CB:invoked when system read a keystroke from ps2 driver.

> 5. **EIP check callback**

DECAF\_EIP\_CHECK\_CB: for every function call, it will invoked this callback before it jump to target function specified by EIP. This is used to check if this EIP is tainted when you specify some taint source. If it’s tainted, it usually means  exploit happens.

> 6. **network callback**
it’s defined in  /shared/DECAF\_callback\_common.h:DECAF\_callback\_type\_t

DECAF\_NIC\_REC\_CB:invoked when network card receives data.

DECAF\_NIC\_SEND\_CB:invoked when network card sends data.

Currently, DECAF’s implementation of network callback is based on NE2000 network card. So, when you write plugins using network callback ,please use NE2000 network card when you start DECAF by specifying “-device ne2k\_pci,netdev=mynet”.



### 1.2 Callback register/unregister function ###

> To use above callback, you should firstly register callback. Before you unload your plugin, please make sure this callback is unregistered. Or it may cause unexpected crash.

> To register/unregister VMI callback types, you should use the following code:

```
DECAF_Handle VMI_register_callback(procmod_callback_type_t cb_type,
                procmod_callback_func_t cb_func,
                int *cb_cond
                );

int VMI_unregister_callback(procmod_callback_type_t cb_type, DECAF_Handle handle);
```

> To register/unregister other callback types, you should use the following code:

```
DECAF_Handle DECAF_register_callback(
		DECAF_callback_type_t cb_type,
		DECAF_callback_func_t cb_func,
		int *cb_cond
                );

int DECAF_unregister_callback(DECAF_callback_type_t cb_type, DECAF_Handle handle);


```

> For block begin/end callback ,you can also use the following for a high performance.

```
DECAF_Handle DECAF_registerOptimizedBlockBeginCallback(
    DECAF_callback_func_t cb_func,
    int *cb_cond,
    gva_t addr,
    OCB_t type);

DECAF_Handle DECAF_registerOptimizedBlockEndCallback(
    DECAF_callback_func_t cb_func,
    int *cb_cond,
    gva_t from,
    gva_t to);

int DECAF_unregisterOptimizedBlockBeginCallback(DECAF_Handle handle);

int DECAF_unregisterOptimizedBlockEndCallback(DECAF_Handle handle);

```

### 1.3 API Hook interfaces ###

> DECAF provides interfaces to hook any api in guest os. You can find the detailed description of these interfaces in **/DECAF/shared/hookapi.h**

```
uintptr_t hookapi_hook_function_byname(const char *mod,const char *func,int is_global,target_ulong cr3,hook_proc_t fnhook,void *opaque,uint32_t sizeof_opaque);

void hookapi_remove_hook(uintptr_t handle);

uintptr_t hookapi_hook_return(target_ulong pc,hook_proc_t fnhook,  void *opaque, uint32_t sizeof_opaque);

uintptr_t hookapi_hook_function(int is_global, target_ulong pc, target_ulong cr3, hook_proc_t fnhook, void *opaque, uint32_t sizeof_opaque );

```

## 2 Utils ##

> As you see in the [sample plugins](http://code.google.com/p/decaf-platform/wiki/decaf_plugins), you registered a **DECAF\_INSN\_BEGIN\_CB** callback, but what can you do in your handler **my\_insn\_begin\_callback** ? The utils interfaces is supposed to help you get more about guest os.You can find these definitions in **DECAF/shared/DECAF\_main.c/h** or **DECAF/shared/vmi.h/vmi\_c\_wrapper.h**. Now,We make a short table for these utils.

  1. Utils for read/write memory from/to guest OS
```
  /*given a virtual address of guest os, get the corresponded physical address */
  gpa_t DECAF_get_phys_addr(CPUState* env, gva_t addr);

  DECAF_errno_t DECAF_memory_rw(CPUState* env, uint32_t addr, void *buf, int len,int is_write);

  DECAF_errno_t DECAF_memory_rw_with_cr3(CPUState* env, target_ulong cr3,gva_t addr, void *buf, int len, int is_write);

  DECAF_errno_t DECAF_read_mem(CPUState* env, gva_t vaddr, int len, void *buf);

  DECAF_errno_t DECAF_write_mem(CPUState* env, gva_t vaddr, int len, void *buf);

  DECAF_errno_t DECAF_read_mem_with_cr3(CPUState* env, target_ulong cr3,gva_t vaddr, int len, void *buf);

  DECAF_errno_t DECAF_write_mem_with_cr3(CPUState* env, target_ulong cr3,gva_t vaddr, int len, void *buf);

  DECAF_get_page_access(CPUState* env, gva_t addr);

```
  1. Utils for get OS-level semantics
VMI component is implemented using c++. You can use the c++ interfaces from vmi.h in your plugin if you can handle c/c++ mix programming well. Also you can use the exported c interfaces defined in vmi\_c\_wrapper.h.

**c++ interfaces**
```

module * VMI_find_module_by_pc(target_ulong pc, target_ulong pgd, target_ulong *base);

module * VMI_find_module_by_name(const char *name, target_ulong pgd, target_ulong *base);

module * VMI_find_module_by_base(target_ulong pgd, uint32_t base);

process * VMI_find_process_by_pid(uint32_t pid);

process * VMI_find_process_by_pgd(uint32_t pgd);

process* VMI_find_process_by_name(char *name);
```

**c interfaces**

```

/// @ingroup semantics
/// locate the module that a given instruction belongs to
/// @param eip virtual address of a given instruction
/// @param cr3 memory space id: physical address of page table
/// @param proc process name (output argument)
/// @param tm return tmodinfo_t structure
extern int   VMI_locate_module_c(gva_t eip, gva_t cr3, char proc[],tmodinfo_t *tm);

//extern int checkcr3(uint32_t cr3, uint32_t eip, uint32_t tracepid, char *name,
  //           int len, uint32_t * offset);

extern int VMI_locate_module_byname_c(const char *name, uint32_t pid,tmodinfo_t * tm);

extern int VMI_find_cr3_by_pid_c(uint32_t pid);

extern int VMI_find_pid_by_cr3_c(uint32_t cr3);

extern int VMI_find_pid_by_name_c(char* proc_name);

/// @ingroup semantics
/// find process given a memory space id
/// @param cr3 memory space id: physical address of page table
/// @param proc process name (output argument)
/// @param pid  process pid (output argument)
/// @return number of modules in this process
extern int VMI_find_process_by_cr3_c(uint32_t cr3, char proc_name[], size_t len, uint32_t *pid);
/* find process name and CR3 using the PID as search key  */
extern int VMI_find_process_by_pid_c(uint32_t pid, char proc_name[], size_t len, uint32_t *cr3);

extern int VMI_get_proc_modules_c(uint32_t pid, uint32_t mod_no, tmodinfo_t *buf);

extern int VMI_get_all_processes_count_c(void);
/* Create array with info about all processes running in system
    */
extern int VMI_find_all_processes_info_c(size_t num_proc, procinfo_t *arr);

//Aravind - added to get the number of loaded modules for the process. This is needed to create the memory required by get_proc_modules
extern int VMI_get_loaded_modules_count_c(uint32_t pid);
//end - Aravind

/// @ingroup semantics
/// @return the current thread id. If for some reason, this operation
/// is not successful, the return value is set to -1.
/// This function only works in Windows XP for Now.
extern int VMI_get_current_tid_c(CPUState* env);

//0 unknown 1 windows 2 linux
extern int VMI_get_guest_version_c(void);

```
