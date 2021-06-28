# Embedded OS Lab5 Fat File System
###### tags: `Embedded OS`

StudentID: N26095025

## Lab5 Descriptions
In this lab, we added a micro SD card module to implement a fat file system on FreeRTOS.

## Development Toolkits
* Board: STM32F407VG
* IDE: STM32CubeIDE 1.5.1
* micro USB adaptor
* 16GB micro SD card
* TTL to USB adaptor


## Directory Hierachy
```
Core/
    +-- Inc/
    |   +-- main.h
    |   +-- AUDIO.h
    |   +-- AUDIO_LINK.h
    |   +-- File_Handling.h
    |   +-- cs43l22.h
    |   +-- diskio.h
    |   +-- fatfs.h
    |   +-- ff.h
    |   +-- ff_gen_drv.h
    |   +-- ffconf.h
    |   +-- fops.h
    |   +-- integer.h
    |   +-- main.h
    |   +-- mmc_sd.h
    |   +-- printf.h
    |   +-- stm32f4xx_hal_conf.h
    |   +-- stm32f4xx_it.h
    |   +-- user_diskio.h
    |   +-- waveplayer.h
    +-- Src/
        +-- main.c
        +-- AUDIO.c
        +-- AUDIO_LINK.c
        +-- File_Handling.c
        +-- cs43l22.c
        +-- diskio.c
        +-- fatfs.c
        +-- ff.c
        +-- ff_gen_drv.c
        +-- fops.c
        +-- main.c
        +-- mmc_sd.c
        +-- user_diskio.c
        ++-- waveplayer.c


FreeRTOS/
    +-- inlcude/*
    +-- portable/
    |   +-- ARM_CM4F/
    |   |   +-- port.c
    |   |   +-- portmacro.h
    |   +-- MemMang/
    |       +-- heap_2.c
    +-- croutine.c
    +-- event_groups.c
    +-- list.c
    +-- queue.c
    +-- stream_buffer.c
    +-- tasks.c
    +-- timers.c
    +-- readme.txt
```

* Make sure your workspace has these files in correct path
    * ```FreeRTOS/``` is cloned from https://github.com/FreeRTOS/FreeRTOS 
        * Add library path ```/FreeRTOS/include```, ```/FreeRTOS/portable/ARM_CM4F```
        * Remmemver to change ```MemMang/``` from ```heap_4.c``` to ```heap_2.c```

## Configurations

1. In file ```FreeRTOSConfig.h```:
    * Change ~~```#ifdef __ICCARM__```~~ to 
    * ```c
        #if defined(__ICCARM__) || defined(__CC_ARM) || defined(__GNUC__)
        ```
    * Set these defines to 0  
    * ```c
        #define configUSE_IDLE_HOOK            0
        #define configUSE_TICK_HOOK            0
        #define configUSE_MALLOC_FAILED_HOOK   0
        #define configCHECK_FOR_STACK_OVERFLOW 0
        //for Lab4
        #define configTOTAL_HEAP_SIZE          ((size_t)(5*1024))
        #define configUSE_TIMERS               0
        ```
2. In file ```stm32f4xx_it.c``` and ```stm32f4xx_it.h```:
    * comment these related functions which will conflict with FreeRTOS's implementations
        * ```PendSV_Handler```
        * ```SVC_Handler```
        * ```SysTick_Handler```

## Source Codes Explanations

### part 1
* Use a semaphore in both task to ensure both of them dont use the file descriptor at the same time.
```clike
void Task1(void *pvParameters) {
  uint8_t count=0;
  for (;;) {
	if (xSemaphore != NULL){
	  FIL test;

	  if ( xSemaphoreTake(xSemaphore, portMAX_DELAY ) == pdTRUE)
	  {
	    if (f_open(&test, "TEST.TXT", FA_WRITE) != 0) printf("open file err in task1\r\n");
	    else printf("file open ok in task1\r\n");

		char buff[13];
		memset(buff, 0, 13);
		if(count%2) sprintf(buff, "%s", "DataWriteT");
		else sprintf(buff, "%s", "DataWriteF");

		UINT byteswrite;
		f_write(&test, buff, 12, &byteswrite);
		f_close(&test);
		count++;

		xSemaphoreGive( xSemaphore );
	  }

    }// if (xSemasphore != NULL)
    vTaskDelay(1);
  }
}

void Task2(void *pvParameters) {
  for (;;) {
    if (xSemaphore != NULL) {

      FIL test_1;

      if ( xSemaphoreTake(xSemaphore, portMAX_DELAY ) == pdTRUE)
      {
	    if (f_open(&test_1, "TEST.TXT", FA_READ) != 0) printf("open file err in task2\r\n");
	    else printf("file open ok in task2\r\n");

	    char buff[11];
		memset(buff, 0, 11);
		UINT bytesread;
		f_read(&test_1, buff, 11, &bytesread);
		printf("data read is %s in task 2\r\n", buff);
		f_close(&test_1);

		xSemaphoreGive( xSemaphore );
	  }

    }//if xSemasphore != NULL
    vTaskDelay(1);
  }
}
```

### part 2
* Use ```AUDIO_PLAYER_Start``` to start the waveplayer
* check the ```AudioState``` at the end of while loop, when it is ```AUDIO_STATE_STOP``` quit the process ```AUDIO_PLAYER_Process```.
```clike=
void Task3(void *pvParameters) {

	TickType_t last_time;
	const TickType_t inv = 5000/ portTICK_PERIOD_MS;
	last_time = xTaskGetTickCount();
	//PlayInit();
	for (;;) {
		AUDIO_PLAYER_Start(0);
		while(!isFinished){
			AUDIO_PLAYER_Process(true);

			if(xTaskGetTickCount() - last_time >= inv){
				AudioState = AUDIO_STATE_NEXT;
				last_time = xTaskGetTickCount();
			}

			if(AudioState == AUDIO_STATE_STOP){
				isFinished =1;
			}
		}
	}

}
```

# Results
* https://youtu.be/pfL6iA7a4lA
