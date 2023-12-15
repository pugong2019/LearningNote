## Malloc Debug

Malloc debug is a method of debugging native memory problems. It can help detect memory corruption, memory leaks, and use after free issues.  
In order to enable malloc debug, you must be able to set special system properties using the setprop command from the shell. This requires the ability to run as **root** on the device.
When malloc debug is enabled, it works by adding a shim layer that replaces
the normal allocation calls. The replaced calls are:

* `malloc`
* `free`
* `calloc`
* `realloc`
* `posix_memalign`
* `memalign`
* `malloc_usable_size`

### Prerequisites

```shell
adb connect device_ip
adb root
adb remount
adb shell setenforce 0
```

### For platform developers
#### Mostly used prop
1. backtrace: enable capturing the backtrace of each allocation sit and the default is 16 frames, the maximumum value this can be set to is 256.
2. leak_track: track all live allocations. When the program terminates, all of the live allocations will be dumped to the log. If the backtrace option was enabled, then the log will include the backtrace of the leaked allocations. 
3. free_track: use after free check
4. verify_pointers: double free check
5. guard: memory corruption check
6. backtrace_dump_on_exit: when the backtrace option has been enabled, this causes the backtrace dump heap data to be dumped to a file when the program exits

#### Use prop to set param:  
Enable backtrace tracking of all allocation for all processes:
```shell
adb shell setprop libc.debug.malloc.options backtrace
adb shell "stop;start"
```
Another prop ***libc.debug.malloc.program*** is to target specific processes under the premise specified by ***libc.debug.malloc.options***.  
For example, enable backtrace tracking for system_server process, that is, malloc_debug only takes effect for the system_server process:

```shell
adb shell setprop libc.debug.malloc.options backtrace
adb shell setprop libc.debug.malloc.program system_server
adb shell "stop;start"
```
Enable backtrace tracking for the zygote and zygote based processes:

```shell
adb shell setprop libc.debug.malloc.program app_process
adb shell setprop libc.debug.malloc.options backtrace
adb shell "stop;start"
```

Enable multiple options in one setup(backtrace, leak_track, guard and backtrace_full):
```shell
adb shell setprop libc.debug.malloc.options "\"backtrace leak_track backtrace_full guard"\"
adb shell "stop;start"
```
Note: the prop has a MAX LENGTH limit(92 bytes), so if when the path is too long or the spcified prop type is too much, adb shell set prop would satisfy the need.
To solve this, we could use environment variable to set the prop.

#### Use environment variable:
Only enable this function in the current shell environment  
```shell
adb shell
celadon_ivi$: export LIBC_DEBUG_MALLOC_OPTIONS=backtrace\ front_guard=16\ rear_guard=16\ backtrace_full\ backtrace_dump_on_exit\ backtrace_dump_prefix=targer_dir
```
Memory leak detection for specific platform process:  
```
adb shell
setprop libc.debug.malloc.program test_malloc_debug
export LIBC_DEBUG_MALLOC_OPTIONS=backtrace\ guard\ leak_track\ free_track\ verbose
```
We use a native test process as a sample(Use ndk to build and push to /system/bin):
```c++
#sample code
#include <string.h>
#include <stdlib.h>
#include <sys/time.h>
#include <unistd.h>
#include <time.h>
#include <vector>
#include <android/log.h>

#define ALOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__);
#define ALOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__);
#define ALOGW(...) __android_log_print(ANDROID_LOG_WARN, LOG_TAG, __VA_ARGS__);
#define ALOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__);

class Foo {
public:
	Foo() {
		arry = new char[1 * 1024 * 1024];
		memset(arry, 0, 1 * 1024 * 1024);
		ALOGD("arry: %p\n", arry);
	};
 
	~Foo() {
		// delete[] arry;
		ALOGD("~Foo(), but not free 1M memory");
	};
	char* arry;
};

using namespace std;
int main(int argc, char* argv[]) { 
	char *ptr = NULL;
	ALOGD("my pid is: %d\n", getpid());
	{
		Foo foo;
	}
	sleep(3);
	ptr = (char*)malloc(10);
	if (!ptr) {
		return -1;
	}
	ALOGD("make memory corruption event: ptr = (char*)malloc(10): %p\n", ptr);
	memset(ptr, 0, 10);

	// front_guard test
	ALOGD("*(ptr - 16) = %d\n", *(ptr - 16));
	*(ptr - 16) = 0xee;

	ALOGD("*(ptr + 10) = %d\n", *(ptr + 10));
	*(ptr + 10) = 0xee;
	
	// free will trigger memory corruption 
	free(ptr);
	ALOGD("memory corruption occuring, please check log \n");

	memset(ptr, 10, 10);
	ALOGD("Test use after free\n");
    return 0;
}
```
When the process exited, the memory leak trace would output into the log like following:
```c
$: adb shell logcat -s malloc_debug
--------- beginning of main
12-13 06:34:07.578 19522 19522 I malloc_debug: test_malloc_debug: Run: 'kill -47 19522' to dump the backtrace.
12-13 06:34:07.578 19522 19522 I malloc_debug: test_malloc_debug: malloc debug enabled
12-13 06:34:10.579 19522 19522 E malloc_debug: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
12-13 06:34:10.579 19522 19522 E malloc_debug: +++ ALLOCATION 0x74c87910bef0 SIZE 10 HAS A CORRUPTED FRONT GUARD
12-13 06:34:10.579 19522 19522 E malloc_debug:   allocation[-16] = 0xee (expected 0xaa)
12-13 06:34:10.579 19522 19522 E malloc_debug: Backtrace at time of failure:
12-13 06:34:10.579 19522 19522 E malloc_debug:           #00  pc 0000000000001a06  /data/test_malloc_debug
12-13 06:34:10.579 19522 19522 E malloc_debug:           #01  pc 000000000004ffd6  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+86)
12-13 06:34:10.579 19522 19522 E malloc_debug: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
12-13 06:34:10.580 19522 19522 E malloc_debug: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
12-13 06:34:10.580 19522 19522 E malloc_debug: +++ ALLOCATION 0x74c87910bef0 USED AFTER FREE
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[0] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[1] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[2] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[3] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[4] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[5] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[6] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[7] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[8] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug:   allocation[9] = 0x0a (expected 0xef)
12-13 06:34:10.580 19522 19522 E malloc_debug: Backtrace at time of free:
12-13 06:34:10.580 19522 19522 E malloc_debug:           #00  pc 0000000000001a06  /data/test_malloc_debug
12-13 06:34:10.580 19522 19522 E malloc_debug:           #01  pc 000000000004ffd6  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+86)
12-13 06:34:10.580 19522 19522 E malloc_debug: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
12-13 06:34:10.580 19522 19522 E malloc_debug: +++ test_malloc_debug leaked block of size 1048576 at 0x74c78660dfe0 (leak 1 of 5)
12-13 06:34:10.580 19522 19522 E malloc_debug: Backtrace at time of allocation:
12-13 06:34:10.580 19522 19522 E malloc_debug:           #00  pc 0000000000045786  /apex/com.android.runtime/lib64/bionic/libc.so (malloc+54)
12-13 06:34:10.580 19522 19522 E malloc_debug:           #01  pc 00000000000b2454  /system/lib64/libc++_shared.so (operator new(unsigned long)+20)
12-13 06:34:10.580 19522 19522 E malloc_debug:           #02  pc 0000000000001934  /data/test_malloc_debug
12-13 06:34:10.580 19522 19522 E malloc_debug:           #03  pc 000000000004ffd6  /apex/com.android.runtime/lib64/bionic/libc.so (__libc_init+86)
```
From the log, we could know that the test_malloc_debug process has memory leak(**leak_track**), memory corruption(**guard**) and memory use after free(**free_track**).

#### Dump heap data
If we want to see the heap data during process is running:
we could use below cmd:
```shell
adb shell
kill -47 target_pid  # Generally saved in /data/local/tmp/backtrace_heap.pid.txt.
```
After the process receives the signal, the backtrace will not be dumped immediately. Second, it will not be triggered until the next call to malloc or free.  
And if we set **backtrace_dump_on_exit**，the heap data in the heap will be dumped when the program exits. Generally saved in /data/local/tmp/backtrace_heap.pid.exit.txt.  
1. The dumptrace file records memory pointers that are not released before the dump is triggered, so not all abnormal memory pointers in the dump file need to be compared carefully.
2. The memory pointer recorded in the dumptrace file is the location of the memory application. You need to check the code to confirm where the pointer is used and where it may not be released.

#### To detect leaks while an app is running:
```shell
adb shell dumpsys meminfo --unreachable <PID_OF_APP>
```
**Without enabling malloc debug, this command will only tell you whether it can detect leaked memory, not where those leaks are occurring**. If you enable malloc debug with the backtrace option for your app before running the dumpsys command, you'll get backtraces showing where the memory was allocated.
For backtraces from your app to be useful, you‘ll want to keep the symbols in your app’s shared libraries rather than stripping them. That way you'll see the location of the leak directly without having to use something like the ndk-stack tool.

### Reference document  
   1. Malloc Debug: https://android.googlesource.com/platform/bionic/+/master/libc/malloc_debug/README.md  
   2. Android 中malloc_debug 使用详解: https://justinwei.blog.csdn.net/article/details/129205767  
