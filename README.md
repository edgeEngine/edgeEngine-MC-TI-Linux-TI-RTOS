# edgeEngine-MC-TI-Linux-TI-RTOS

## Hybrid edge cloud binaries for j721e (TI Jacinto 7 processor platform)

## Before you start  

 [explore the developer resources & documentation](https://developer.mimik.com)
 
 [sign up and create a mimik developer console account](https://developer.mimik.com/console/create_account)


## About:

### Overview

This repository provides a pair of ELF objects -- an edge main as executable
and an edgechild (as static library) for Texas Instruments Jacinto 7 platform: j721e.

edge main running in A72 Linux dynamically deploys and pass on -- as requested
by user --, wasm microservice requests to edgechild running in R5F rtos at core
MCU2_0. edgechild, upon receiving the image or using previous deployed image,
will execute the wasm microservice in R5F rtos. Both edge parent in A72 Linux
and edgechild in R5F uses low-level TI inter processor communication facility
(IPC) to exchange wasm image data and mimik RPC protocol packets. edgechild in
R5F and edge parent in A72 uses IPC port 16 for communication.

This repository contains the following:

**edge** : A binary to be executed on A72 Linux of TI Jacinto 7 - j721e

**edgechild.lib** : A static library containing required binary objects to be used by applications
running on RTOS R5F core(MCU2_0) of TI Jacinto 7 - j721e platform

**mimik_main.h** : A c header file to be included in application code wishing to make use of edgechild.lib

## edgeChild-ti-rtos
edgeChild for TI RTOS Platform

We have given below, required  steps to use edgeChild along with TI vision_apps. To Prepare environment, compile and running it, please read TI [document](https://software-dl.ti.com/jacinto7/esd/processor-sdk-rtos-jacinto7/latest/exports/docs/vision_apps/docs/user_guide/BUILD_AND_RUN.html).

### Building instructions

**1.** Download the latest edgeChild-ti-rtos release from [HERE](https://github.com/edgeEngine/edgeEngine-MC-TI-Linux-TI-RTOS/releases)

**2.** Untar the package using tar utility as follows.

```shell
$ tar xvf edgeChild-ti-rtos-v1.0.0.tar
```

**3.** set up the environment variables for TI psdk rtos path and psdk linux path as follows.

```shell
$ export PSDKR_PATH=/home/user1/jacinto/ti-processor-sdk-rtos-j721e-evm-08_02_00_05
$ export PSDKL_PATH=/home/user1/jacinto/ti-processor-sdk-linux-j7-evm-08_02_00_03
```

#### 4. Steps to run mimik edge child in rtos R5F ( mcu2_0 core ) along with TI vison_apps 

The following steps provide a way to modify TI vision_apps mcu2_0 code to
include our edgechild.lib and there by enale running edgechild along with TI
vision_apps example application upon installation.

**Step1:** cd to $PSDKR_PATH/vision_apps/platform/j721e/rtos/mcu2_0 to make all the needed changes

```shell
cd /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps/platform/j721e/rtos/mcu2_0
```

**Step2:** copy edgechild.lib and mimik_main.h to mcu2_0 directory

```shell
  cp mimik_main.h /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps/platform/j721e/rtos/mcu2_0/
  cp edgechild.lib /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps/platform/j721e/rtos/mcu2_0/
```

**Step3:** Edit $PSDKR_PATH/vision_apps/platform/j721e/rtos/mcu2_0/concerto.mak to add edgechild.lib

Edit mcu2_0/concerto.mak and add the following lines for the definition of
mimik_main() by including edgechild.lib
(Look at the diff changes given below to help with making the changes easily)

LDIRS += $(VISION_APPS_PATH)/platform/$(SOC)/rtos/mcu2_0/
STATIC_LIBS += edgechild

```shell
gedit /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps/platform/j721e/rtos/mcu2_0/concerto.mak
```

**Step4:** Edit $PSDKR_PATH/vision_apps/platform/j721e/rtos/mcu2_0/linker.cmd

To run edge-child and to dynamically receive and execute wasm microservice bytecode image in R5F, we
need more stack and heap size than the default used by vision_apps in mcu2_0 R5F core.
Hence edit linker.cmd and make the changes 

(Look at the diff changes given below to help with making the changes easily)

change from: ---stack_size=0x2000 to: --stack_size=0x10000
change from: ---heap_size=0x1000 to: --heap_size=0x10000

```shell
gedit /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps/platform/j721e/rtos/mcu2_0/linker.cmd
```

**Step5:** Edit $PSDKR_PATH/vision_apps/platform/j721e/rtos/mcu2_0/main.c

Edit mcu2_0/main.c add the following lines to call the mimik_main() and to
invoke edge child inside function appMain()

(Look at the diff changes given below to help with making the changes easily)

NOTE: mimik_main() should be called after the call to appInit(); because
mimik_main() needs prior proper initialization of IPC and TI board.  We did not
add any TI initializations inside mimik_main() as each app developer may want
to initialize based on their needs.

Also mimik_main() needs a larger stack and hence change gTskStackMain to 64K

static uint8_t gTskStackMain[8*1024] to 64K static uint8_t gTskStackMain[64*1024]

#include "mimik_main.h"
mimik_main();
static uint8_t gTskStackMain[64*1024]

```shell
gedit /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps/platform/j721e/rtos/mcu2_0/main.c
```

**Step6:** To clean build $PSDKR_PATH/vision_apps/

cd to $PSDKR_PATH/vision_apps/  and clean old objects - remove old output and
libs folder

```shell
cd /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps
make clean
rm -rf out
rm -rf lib
```

**step7:** Build vision_apps including the edgechild runtime for rtos mcu2_0

```shell
make vision_apps -j3
```
**Step8:** Review the make logs to ensure there are no errors.

To confirm if muu2_0 got built, you could check

```shell
ls -l /home/user1/TiJacinto7/ti-processor-sdk-rtos-j721e-evm-08_02_00_05/vision_apps/out/J7/R5F/FREERTOS/release/app_rtos_common_mcu2_0.lib
```

---------------------------------------------------------------------------------------------------------------------
#### diff output comparing original vision_apps mcu2_0 with modified mcu2_0 using edgechild.lib

```c
diff -up mcu2_0.original/concerto.mak mcu2_0.edgechild/concerto.mak
--- mcu2_0.original/concerto.mak	2022-10-14 11:05:07.798713353 -0700
+++ mcu2_0.edgechild/concerto.mak	2022-10-14 11:11:23.280213868 -0700
@@ -18,6 +18,9 @@ IDIRS+=$(VISION_APPS_PATH)/platform/$(SO
 
 STATIC_LIBS += app_rtos_linux
 
+LDIRS += $(VISION_APPS_PATH)/platform/$(SOC)/rtos/mcu2_0/
+STATIC_LIBS += edgechild
+
 include $(FINALE)
 
 endif
Only in mcu2_0.edgechild: edgechild.lib
diff -up mcu2_0.original/linker.cmd mcu2_0.edgechild/linker.cmd
--- mcu2_0.original/linker.cmd	2022-10-14 11:05:07.798713353 -0700
+++ mcu2_0.edgechild/linker.cmd	2022-10-14 11:08:58.667955599 -0700
@@ -1,7 +1,7 @@
 /* linker options */
 --fill_value=0
---stack_size=0x2000
---heap_size=0x1000
+--stack_size=0x10000
+--heap_size=0x10000
 
 #define ATCM_START 0x00000000
 
diff -up mcu2_0.original/main.c mcu2_0.edgechild/main.c
--- mcu2_0.original/main.c	2022-10-14 11:05:07.798713353 -0700
+++ mcu2_0.edgechild/main.c	2022-10-14 11:10:15.760133652 -0700
@@ -70,11 +70,13 @@
 #include <ti/osal/TaskP.h>
 #include <app_ipc_rsctable.h>
 #include "app_cfg_mcu2_0.h"
+#include "mimik_main.h"
 
 static void appMain(void* arg0, void* arg1)
 {
     appUtilsTaskInit();
     appInit();
+    mimik_main();    
     appRun();
     #if 1
     while(1)
@@ -94,7 +96,7 @@ void StartupEmulatorWaitFxn (void)
     }while (enableDebug);
 }
 
-static uint8_t gTskStackMain[8*1024]
+static uint8_t gTskStackMain[64*1024]
 __attribute__ ((section(".bss:taskStackSection")))
 __attribute__ ((aligned(8192)))
     ;
Only in mcu2_0.edgechild: mimik_main.h
```

---------------------------------------------------------------------------------------------------------------------

**5.** Install and run vision_apps with edgechild

Insert the prepared vision_apps sdcard into Linux machine, and from vision_apps directory
install vision_apps with the edgechild for rtos as the follows

```shell
$ make linux_fs_install_sd
```

Plugin the sdcard in to the Ti Jacinto 7 j721e board, and power up the board, the bootloader will load the mcu image and run it, -- along with edgechild.

**6.** To view the execution logs of processors from Linux A72 

To see the applications log, login to j721e linux as follows, replacing the ip
correctly instead of example 10.0.0.96

```shell
$ ssh root@10.0.0.96
$ cd /opt/vision_apps
$ ../vision_apps_init.sh
```

It shows all execution logs of the cpu, mcu, and dsp processors

## edgeEngine-MC-TI-Linux-TI-RTOS

edgeEngine for TI Linux Platform

### Installation Guide
**1.** Download latest release [HERE](https://github.com/edgeEngine/edgeEngine-MC-TI-Linux-TI-RTOS/releases)
**2.** Create a new directory
**3.** Move package to newly created directory 
**4.** Open terminal and navigate to the newly created directory that now has the downloaded .tar file
**5.** *Untar package (ex:)
```shell
tar xvf edgeEngine-ti-linux-v3.0.0.tar
```
**6.** Run start script to start edgeEngine
```
./start.sh
```
**7.** Please visit [Developer Console](https://developer.mimik.com/console/create_account) to create an account and get started with your projects

#### NOTE:
- A directory may be made after untaring. Navigate into that directory to find `start.sh` script 
- Do not close terminal window where edgeEngine is running. Closing this window will terminate edgeEngine process
- To stop edgeEngine simply close or use keyboard shortcut CTRL + C in terminal window where edgeEngine is running
