# Tutorials on Developing Plugins #
## 1. Introduction ##
DECAF provides many callback interfaces for developers to write powerful plugins to instrument the execution of the guest operating system.  It invokes the callback at runtime,so you can dynamically enable or disable,register or unregister callbacks. With these callback interfaces, you can retrieve OS-level semantics including process ,system api,keystroke,network,etc, completely  outside  the guest operating system.This manual provides some basic knowledge on developing plugins for DECAF.

## 2. Sample plugin ##

The following code is a sample plugin. With this plugin, you can specify a process you wanna trace and print out the process name when the process starts. It's small but complete.

After DECAF loads the plugin, it will call **init\_plugin(void)** first. You can use **plugin\_interface\_t**(DECAF\_main.h) to define the behaviors of plugins. But the most important thing is to specify the **plugin\_cleanup** interface.We usually free the resources allocated for this plugin using this interface.If we do not do this or do not free resources completely, DECAF may crash.Another thing you can do using **plugin\_interface\_t** is to define your own command for DECAF. So you can interact with your plugin at runtime.

```
  #include "DECAF_types.h"
  #include "DECAF_main.h"
  #include "DECAF_callback.h"
  #include "DECAF_callback_common.h"
  #include "vmi_callback.h"
  #include "utils/Output.h"
  #include "DECAF_target.h"
  
  //basic stub for plugins
  static plugin_interface_t my_interface;
  static DECAF_Handle processbegin_handle = DECAF_NULL_HANDLE;
  
  char targetname[512];
  
  /*
  * This callback is invoked when a new process starts in the guest OS.
  */
  static void my_loadmainmodule_callback(VMI_Callback_Params* params)
  {
    if(strcmp(params->cp.name,targetname)==0)
  	  DECAF_printf("Process %s you spcecified starts \n",params->cp.name);
  }
  /*
   * Handler to implement the command monitor_proc.
   */
  
  void do_monitor_proc(Monitor* mon, const QDict* qdict)
  {
  	/*
  	 * Copy the name of the process to be monitored to targetname.
  	 */
    if ((qdict != NULL) && (qdict_haskey(qdict, "procname"))) {
      strncpy(targetname, qdict_get_str(qdict, "procname"), 512);
    }
  targetname[511] = '\0';
  }
  
  static int my_init(void)
  {
    DECAF_printf("Hello World\n");
  
    //register for process create and process remove events  
    processbegin_handle = VMI_register_callback(VMI_CREATEPROC_CB, &my_loadmainmodule_callback, NULL);
    if (processbegin_handle == DECAF_NULL_HANDLE)
    {
      DECAF_printf("Could not register for the create or remove proc events\n");  
    }
    return (0);
  }
  
  /*
   * This function is invoked when the plugin is unloaded.
   */
  static void my_cleanup(void)
  {
    DECAF_printf("Bye world\n");
    /*
     * Unregister for the process start and exit callbacks.
     */
    if (processbegin_handle != DECAF_NULL_HANDLE) {
      VMI_unregister_callback(VMI_CREATEPROC_CB, processbegin_handle);  
      processbegin_handle = DECAF_NULL_HANDLE;
    }
    } 
  /*
   * Commands supported by the plugin. Included in plugin_cmds.h
   */
  static mon_cmd_t my_term_cmds[] = {
    		{
  			.name		= "monitor_proc",
  			.args_type	= "procname:s?",
  			.mhandler.cmd	= do_monitor_proc,
  			.params		= "[procname]",
  			.help		= "Run the tests with program [procname]"
  		},
    {NULL, NULL, },
  };
  
  /*
   * This function registers the plugin_interface with DECAF.
   * The interface is used to register custom commands, let DECAF know which cleanup function to call upon plugin unload, etc,.
   */
  plugin_interface_t* init_plugin(void)
  {
    my_interface.mon_cmds = my_term_cmds;
    my_interface.plugin_cleanup = &my_cleanup;
    
    my_init();
    return (&my_interface);
  }
```

Now,you know the **plugin\_cleanup** and **mon\_cmds**. Let's talk about the sample plugin code. As you see, in the function **init\_plugin(void)**, we define our own command using **my\_term\_cmds**. In **my\_term\_cmds**, we specify the command name,command handler,command parameters and help message. You can define whatever you like following this convention. The **plugin\_cleanup** is also defined by **my\_cleanup**. Using **my\_cleanup**, DECAF cleanup all the resources used by the plugin.

In **my\_init()**, we register  **VMI\_CREATEPROC\_CB** callback and its handler **my\_loadmainmodule\_callback**. So when a process starts, DECAF will call **my\_loadmainmodule\_callback**. And from its parameters, you can get the pid,name and cr3 of the process. So you can check if it's the process you specified with monitor\_proc command. If so, we print out the process name. Of course, we can do many other things.For example, we can register **DECAF\_INSN\_BEGIN\_CB** callback and its handler at this place. And in its handler, we check if it belongs to the specified process and print out the instruction's EIP using the following code.
```
  uint32_t target_cr3;
  static void my_insn_begin_callback(DECAF_Callback_Params* params)
  {
       if(params->ib.env->cr[3]==target_cr3)
	{
		DECAF_printf("EIP 0x%08x \n",params->ib.env->eip);
	}
  }
  /*
  * This callback is invoked when a new process starts in the guest OS.
  */
  static void my_loadmainmodule_callback(VMI_Callback_Params* params)
  {
    if(strcmp(params->cp.name,targetname)==0){
  	  DECAF_printf("Process %s you spcecified starts \n",params->cp.name);
    target_cr3=params->cp.cr3;  
     DECAF_register_callback(DECAF_INSN_BEGIN_CB,&my_insn_begin_callback,NULL)
    }
  }


```

One thing you need to take care of is the third parameter of **VMI\_register\_callback**. If it's NULL, it means this callback will be invoked all the time. If it's a pointer pointing to 1, this callback will be invoked all the time. If it's a pointer pointing to 0, this callback is disabled. Other kind of callback registration function is also following this convention.

Now you see how to register or unregister ,enable or disable callbacks. As to the interfaces exported by DECAF to instrument the system,please see [DECAF interfaces](https://code.google.com/p/decaf-platform/wiki/decaf_interfaces)

### 3 Hook api ###

For malware analysis, api trace is essential to understand the behavior of malware. Traditional analysis tool implements api trace by hooking api at different levels which is easy to bypass especially for Rootkit. DECAF can trace api more reliably  outside VM. You can hook any api in your plugins or just hook some EIP.In the plugin [hookapitest](http://code.google.com/p/decaf-platform/downloads/detail?name=decaf_plugins.tar.gz), it shows how to hook a api and how to retrieve api parameters.


Now, we modify the above sample code to hook api NtCreateFile and retrieve it's parameter from the stack.First we register the api hook by **hookapi\_hook\_function\_byname** when target process starts(when **my\_loadmainmodule\_callback** is called). In the following code, NtCreateFile\_call will be invoked when guest os call NtCreateFile. For the parameters marked "IN", you can retrieve them from stack at this place.But for the "OUT" parameters, it's filled when NtCreateFile returns. In order to deal with this case, we hook the api return by using **hookapi\_hook\_return**. When NtCreateFile\_call is invoked, the return address of NtCreateFile is stored on [EBP](EBP.md). We should pass this return value to **hookapi\_hook\_return** function. Also, you can pass data to NtCreateFile\_ret using 3th and 4th parameters of **hookapi\_hook\_return**.

In NtCreateFile\_ret , we retrieve FileHandle form the stack. At this return point, EBP stores the address of first parameters FileHandle. We can use DECAF\_read\_mem to read this FileHandle from [EBP](EBP.md). This is what we do in the following code. You can find more complex parameter retrieving  function in **hookapitests/custom\_handlers.c**. But the basic idea is same.

The key to retrieve parameters correctly is to understand "**address**" correctly. There are three kind of address--guest os's virtual address, guest os's physical address and host os's virtual address. The value of EBP is the guest os's virtual address. DECAF\_read\_mem get the content of specified virtual address stored in guest OS memory. Sometimes, the parameters of API is a pointer, you need to read this pointer's value first, and then read the memory pointed to by this pointer using DECAF\_read\_mem. Another thing you need to take care of is the character set problem. Windows uses Unicode string internally. If you get some unreadable code, you may need to convert it to readable character set. Keep that in mind!!


```

  DECAF_handle ntcreatefile_handle;

typedef struct {
	uint32_t call_stack[12]; //paramters and return address
	DECAF_Handle hook_handle;
} NtCreateFile_hook_context_t;

/*
NTSTATUS NtCreateFile(
  _Out_     PHANDLE FileHandle,
  _In_      ACCESS_MASK DesiredAccess,
  _In_      POBJECT_ATTRIBUTES ObjectAttributes,
  _Out_     PIO_STATUS_BLOCK IoStatusBlock,
  _In_opt_  PLARGE_INTEGER AllocationSize,
  _In_      ULONG FileAttributes,
  _In_      ULONG ShareAccess,
  _In_      ULONG CreateDisposition,
  _In_      ULONG CreateOptions,
  _In_      PVOID EaBuffer,
  _In_      ULONG EaLength
);
*/
static void NtCreateFile_ret(void *param)
{
	NtCreateFile_hook_context_t *ctx = (NtCreateFile_hook_context_t *)param;
	DECAF_printf("NtCreateFile exit:");

	hookapi_remove_hook(ctx->hook_handle);
	uint32_t out_handle;

	DECAF_read_mem(NULL, ctx->call_stack[1], 4, &out_handle);
	DECAF_printf("out_handle=%08x\n", out_handle);
	free(ctx);
}

static void NtCreateFile_call(void *opaque)
{
	DECAF_printf("NtCreateFile entry\n");
	NtCreateFile_hook_context_t *ctx = (NtCreateFile_hook_context_t*)
			malloc(sizeof(NtCreateFile_hook_context_t));
	if(!ctx) //run out of memory
		return;

	DECAF_read_mem(NULL, cpu_single_env->regs[R_ESP], 12*4, ctx->call_stack);
	ctx->hook_handle = hookapi_hook_return(ctx->call_stack[0],
			NtCreateFile_ret, ctx, sizeof(*ctx));
}

  static void my_loadmainmodule_callback(VMI_Callback_Params* params)
  {
    if(strcmp(params->cp.name,targetname)==0){
  	  DECAF_printf("Process %s you spcecified starts \n",params->cp.name);
    target_cr3=params->cp.cr3;  
     
  
/// @ingroup hookapi
/// install a hook at the function entry by specifying module name and function name
/// @param mod module name that this function is located in
/// @param func function name
/// @param is_global flag specifies if this hook should be invoked globally or only in certain execution context (when should_monitor is true)
/// @param cr3 the memory space that this hook is installed. 0 for all memory spaces.
/// @param fnhook address of function hook
/// @param opaque address of an opaque structure provided by caller (has to be globally allocated)
//  @param sizeof_opaque size of the opaque structure (if opaque is an integer, not a pointer to a structure, sizeof_opaque must be zero) 
/// @return a handle that uniquely identifies this hook
/// Note that the handle that is returned, might not actually be active yet - you can check the eip value of the handle to find out
/// the default value is 0.
   ntcreatefile_handle = hookapi_hook_function_byname(
			"ntdll.dll", "NtCreateFile", 1, target_cr3, NtCreateFile_call, NULL, 0);
    }
  }




```

## 4.Tainting ##
DECAF has **bitwise** tainting functionality built in. It’s implemented by inserting taint propagation opcodes into the stream of Tiny Code Generator (TCG) opcodes being executed by the emulator. Because of this, taint propagation within DECAF incurs a much smaller runtime performance penalty than other whole-system dynamic analysis platforms. It’s very easy to develop data flow tracking tools using this functionality.

To enable tainting, you should compile DECAF with “--enable-tcg-taint”. After you start DECAF, the tainting is enabled by default. You can disable it to clear the taint data using command “disable-taint”. Also, you can check the taint information using command “tainted\_bytes”.  One thing you can configure tainting is the pointer tainting. The command “taint\_pointers on/off on/off” turns on/off the pointers read/store tainting.
```
taintcheck_check_virtmem(uint32_t vaddr, uint32_t size, uint8_t * taint)
taintcheck_register_check(int regid,int offset,int size,CPUState *env)
taint_mem(uint32_t addr,int size,uint8_t *taint)
```
Using the above apis exported by DECAF, you can taint a memory or check the taint status of specific memoery or register. To taint a register, just directly set the shadow register taint\_regs defined in the struct CPUState. For example, to taint eax, we can using the following statement.
```
cpu_single_env->taint_regs[R_EAX] = taint_value;//bitwise tainting.

```
DECAF also supports keystroke and network tainting. We include a [keystroke tainting sample](http://decaf-platform.googlecode.com/files/decaf_plugins1.1.tar.gz) in the plugins download. It’s similar to taint network input/output. In the future, we will provide other tainting interface for other hardwares(e.g., disk).



## 5. Configure&Makefile ##

We suggest you use configure/Makefile in the sample plugin we published in [decaf\_plugins.tar.gz](http://decaf-platform.googlecode.com/files/decaf_plugins1.1.tar.gz) as a template to write your own configure/makefile.