# uboot源码分析第2阶段

uboot的基本目标：启动内核。做了什么事？

1. 从Flash读出内核：能支持读写Flash
    + NorFlash：flash_init
    + NandFlash：nand_init
2. 启动它

# start_armboot函数

## gd全局变量

在第1阶段预留的全局数据的内存中，创建了gd全局变量

```c
// 全局数据的结构体，最上面保存的是板级数据
typedef	struct	global_data {
	bd_t		*bd;
	unsigned long	flags;
	unsigned long	baudrate;
	unsigned long	have_console;	/* serial_init() was called */
	unsigned long	reloc_off;	/* Relocation Offset */
	unsigned long	env_addr;	/* Address  of Environment struct */
	unsigned long	env_valid;	/* Checksum of Environment valid? */
	unsigned long	fb_base;	/* base address of frame buffer */
#ifdef CONFIG_VFD
	unsigned char	vfd_type;	/* display type */
#endif
	void		**jt;		/* jump table */
} gd_t;
// 定义了gc类型的指针，为了访问效率，直接保存在寄存器r8中
#define DECLARE_GLOBAL_DATA_PTR     register volatile gd_t *gd asm ("r8")

// 接下来是申请内存，正好就是把第1阶段预留的一段内存拿来使用，并清空
gd = (gd_t*)(_armboot_start - CFG_MALLOC_LEN - sizeof(gd_t));
/* compiler optimization barrier needed for GCC >= 3.4 */
__asm__ __volatile__("": : :"memory");

memset ((void*)gd, 0, sizeof (gd_t));
gd->bd = (bd_t*)((char*)gd - sizeof(bd_t));
memset (gd->bd, 0, sizeof (bd_t));
```

## 一系列初始化

接下来把要初始化的软硬件放在一个函数指针数组中，挨个来进行初始化。我们一个个看看：

+ cpu_init : CPU相关的，位于`cpu/arm920t/cpu.c`
+ board_init : Board相关的，位于`board/100ask24x0.c`
+ interrupt_init : 中断相关的，这肯定跟具体的CPU芯片型号相关，位于`cpu/arm920t/s3c24x0/interrupt.c`
+ env_init : 环境变量初始化。这个初始化的是什么？实际上是环境变量的地址。
实际上，env_init这个函数在多个文件中都有定义，如下分析：

    `env_flash.c`

    如果定义了CFG_ENV_IS_IN_FLASH宏，就会把env_flash.c中的初始化函数编译进去。

    ```c
    #if defined(CFG_ENV_IS_IN_FLASH)
    int  env_init(void)
    {
        gd->env_addr  = (ulong)&default_environment[0];
        gd->env_valid = 0;
        return (0);
    }
    #endif /* CFG_ENV_IS_IN_FLASH */
    ```

    `env_nand.c`

    如果定义了CFG_ENV_IS_IN_NAND宏，就会把env_nand.c中的初始化函数编译进去。

    ```c
    #if defined(CFG_ENV_IS_IN_NAND)
    int env_init(void)
    {
        gd->env_addr  = (ulong)&default_environment[0];
        gd->env_valid = 1;
        return (0);
    }
    #endif /* CFG_ENV_IS_IN_NAND */
    ```

    可以看到，env_init实际上依赖于我们在`include/config/100ask24x0.h`配置文件中设置的宏定义，来选择具体是那个函数来进入编译。当前配置文件的选择是：

    `include/config/100ask24x0.h`配置文件中，定义了保存环境变量的硬件介质和地址

    ```c
    //#define	CFG_ENV_IS_IN_FLASH	1
    #define CFG_ENV_IS_IN_NAND  1
    #define CFG_ENV_OFFSET      0x40000
    #define CFG_ENV_SIZE		0x20000	/* Total Size of Environment Sector */
    ```
+ init_baudrate : 从环境变量中读出波特率。如果没有或为0，就使用配置的默认波特率

    `board.c`

    ```c
    static int init_baudrate (void)
    {
        char tmp[64];	/* long enough for environment variables */
        int i = getenv_r ("baudrate", tmp, sizeof (tmp));
        gd->bd->bi_baudrate = gd->baudrate = (i > 0)
                ? (int) simple_strtoul (tmp, NULL, 10)
                : CONFIG_BAUDRATE;

        return (0);
    }
    ```

    `include/config/100ask24x0.h`配置文件中，定义了默认波特率

    ```c
    #define CONFIG_BAUDRATE		115200
    ```
+ serial_init : 串口初始化。前面已经获取到波特率了，接下来是初始化串口。串口肯定跟CPU的具体型号相关。同时，还要配置选择具体使用哪一个串口

    `cpu/arm920t/s3c24x0/serial.c`，串口初始化的代码。根据配置的CONFIG_SERIALx来选择具体的串口号，然后从gd指针中读取波特率，并进行初始化

    ```c
    #ifdef CONFIG_SERIAL1
    #define UART_NR	S3C24X0_UART0

    #elif defined(CONFIG_SERIAL2)
    # if defined(CONFIG_TRAB)
    #  error "TRAB supports only CONFIG_SERIAL1"
    # endif
    #define UART_NR	S3C24X0_UART1

    #elif defined(CONFIG_SERIAL3)
    # if defined(CONFIG_TRAB)
    #  #error "TRAB supports only CONFIG_SERIAL1"
    # endif
    #define UART_NR	S3C24X0_UART2

    #else
    #error "Bad: you didn't configure serial ..."
    #endif

    void serial_setbrg (void)
    {
        S3C24X0_UART * const uart = S3C24X0_GetBase_UART(UART_NR);
        int i;
        unsigned int reg = 0;

        /* value is calculated so : (int)(PCLK/16./baudrate) -1 */
        reg = get_PCLK() / (16 * gd->baudrate) - 1;

        /* FIFO enable, Tx/Rx FIFO clear */
        uart->UFCON = 0x07;
        uart->UMCON = 0x0;
        /* Normal,No parity,1 stop,8 bit */
        uart->ULCON = 0x3;
        /*
        * tx=level,rx=edge,disable timeout int.,enable rx error int.,
        * normal,interrupt or polling
        */
        uart->UCON = 0x245;
        uart->UBRDIV = reg;

    #ifdef CONFIG_HWFLOW
        uart->UMCON = 0x1; /* RTS up */
    #endif
        for (i = 0; i < 100; i++);
    }

    int serial_init (void)
    {
        serial_setbrg ();

        return (0);
    }
    ```

    `include/config/100ask24x0.h`配置文件中，定义了要使用的串口

    ```c
    /*
    * select serial console configuration
    */
    #define CONFIG_SERIAL1          1	/* we use SERIAL 1 on SMDK2410 */
    ```
+ console_init_f : 设置标记位

    `common/console.c`

    ```c
    int console_init_f (void)
    {
        gd->have_console = 1;

    #ifdef CONFIG_SILENT_CONSOLE
        if (getenv("silent") != NULL)
            gd->flags |= GD_FLG_SILENT;
    #endif

        return (0);
    }
    ```
+ display_banner : 打印uboot信息日志

    ```c
    static int display_banner (void)
    {
        printf ("\n\n%s\n\n", version_string);
        debug ("U-Boot code: %08lX -> %08lX  BSS: -> %08lX\n",
            _armboot_start, _bss_start, _bss_end);
    #ifdef CONFIG_MODEM_SUPPORT
        debug ("Modem Support enabled\n");
    #endif
    #ifdef CONFIG_USE_IRQ
        debug ("IRQ Stack: %08lx\n", IRQ_STACK_START);
        debug ("FIQ Stack: %08lx\n", FIQ_STACK_START);
    #endif

        return (0);
    }
    ```
+ dram_init : 把配置文件中SDRAM的内存地址和大小，传递给gd全局指针，供内核使用。这个每个板子都不一样，所以在board中实现

    `include/config/100ask24x0.h`配置文件中，定义了SDRAM的起始地址和大小

    ```c
    #define PHYS_SDRAM_1		0x30000000 /* SDRAM Bank #1 */
    #define PHYS_SDRAM_1_SIZE	0x04000000 /* 64 MB */
    ```

    `board/100ask24x0.c`中，把SDRAM的信息传递给gd数组

    ```c
    int dram_init (void)
    {
        gd->bd->bi_dram[0].start = PHYS_SDRAM_1;
        gd->bd->bi_dram[0].size = PHYS_SDRAM_1_SIZE;

        return 0;
    }
    ```
+ display_dram_config : 打印SDRAM的内存配置

    `include/config/100ask24x0.h`配置文件中，定义了SDRAM的个数

    ```c
    #define CONFIG_NR_DRAM_BANKS	1	   /* we have 1 bank of DRAM */
    ```

    打印所有内存块的地址和大小

    ```c
    static int display_dram_config (void)
    {
        int i;

    #ifdef DEBUG
        puts ("RAM Configuration:\n");

        for(i=0; i<CONFIG_NR_DRAM_BANKS; i++) {
            printf ("Bank #%d: %08lx ", i, gd->bd->bi_dram[i].start);
            print_size (gd->bd->bi_dram[i].size, "\n");
        }
    #else
        ulong size = 0;

        for (i=0; i<CONFIG_NR_DRAM_BANKS; i++) {
            size += gd->bd->bi_dram[i].size;
        }
        puts("DRAM:  ");
        print_size(size, "\n");
    #endif

        return (0);
    }
    ```

下面是全部的初始化代码：

```c
init_fnc_t *init_sequence[] = {
	cpu_init,		/* basic cpu dependent setup */
	board_init,		/* basic board dependent setup */
	interrupt_init,		/* set up exceptions */
	env_init,		/* initialize environment */
	init_baudrate,		/* initialze baudrate settings */
	serial_init,		/* serial communications setup */
	console_init_f,		/* stage 1 init of console */
	display_banner,		/* say that we are here */
	dram_init,		/* configure available RAM banks */
	display_dram_config,
	NULL,
};

for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
    if ((*init_fnc_ptr)() != 0) {
        hang ();
    }
}
```

## 初始化Flash并打印

如果没有定义`CFG_NO_FLASH`宏，就初始化Flash并打印

```c
#ifndef CFG_NO_FLASH
	/* configure available FLASH banks */
	size = flash_init ();
	display_flash_config (size);
#endif /* CFG_NO_FLASH */
```

## 初始化动态内存sbrk

[探索计算机系统中的动态内存分配：揭秘`sbrk`函数的奥秘](https://blog.csdn.net/m0_72410588/article/details/132241444)

sbrk函数是一种用于在进程地址空间中调整堆空间大小的系统调用。它通过移动堆的边界来增加或减少可用内存的大小。sbrk函数通常用于实现malloc和free等动态内存分配函数，是一种常见的内存管理方式。

```c
/*
 * Begin and End of memory area for malloc(), and current "brk"
 */
static ulong mem_malloc_start = 0;
static ulong mem_malloc_end = 0;
static ulong mem_malloc_brk = 0;

static
void mem_malloc_init (ulong dest_addr)
{
	mem_malloc_start = dest_addr;
	mem_malloc_end = dest_addr + CFG_MALLOC_LEN;
	mem_malloc_brk = mem_malloc_start;

	memset ((void *) mem_malloc_start, 0,
			mem_malloc_end - mem_malloc_start);
}

void *sbrk (ptrdiff_t increment)
{
	ulong old = mem_malloc_brk;
	ulong new = old + increment;

	if ((new < mem_malloc_start) || (new > mem_malloc_end)) {
		return (NULL);
	}
	mem_malloc_brk = new;

	return ((void *) old);
}

/* armboot_start is defined in the board-specific linker script */
mem_malloc_init (_armboot_start - CFG_MALLOC_LEN);
```

sbrk函数的应用举例：

```c
#include <unistd.h>

void *my_malloc(int size) {
    static void *base = NULL;
    void *result;

    if (base == NULL) {
        base = sbrk(0);
    }

    result = sbrk(size);
    if (result == (void *)-1) {
        return NULL;
    }

    return result;
}

int main() {
    int *ptr = my_malloc(sizeof(int));
    if (ptr != NULL) {
        *ptr = 42;
    }
    return 0;
}
```

sbrk函数有以下注意事项：

1. sbrk函数在多线程环境中并不是线程安全的。如果多个线程同时调用sbrk函数，可能会导致内存分配的冲突和错误

2. 使用sbrk函数进行动态内存分配时，需要特别注意内存的释放和回收，避免内存泄漏和野指针等问题

3. 在现代操作系统中，sbrk函数通常已被更高级别的内存分配接口替代，如malloc和free等

