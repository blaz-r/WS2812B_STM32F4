# LowMEM ws2812b library ( STM32F4_WS2812B )
Library forked from https://github.com/hubmartin/WS2812B_STM32F4 and changed a bit to work with blackpill (stm32f411). I also added support for different orders of RGB configuration (GRB, BRG ...). Below is the info from original version of library, it works in almost the same way, just visInit function now takes order of colors ("RGB", "GRB" ...).

## Original info:

This is a memory and CPU efficient implementation of WS2812B library for STM32 processors. **You have to compile it with -Og or at least -O1 optimizations to take advantage of it.**


**See my other repositories for F0 and F3 port.**

The example is implemented for STM32F4 line with TIM1 and DMA2. The DMA2 is necessary because only this DMA has access to the AHB1 bus where the GPIO peripheral is located.
Project is made in Atollic TrueStudio but you can compile it with any ARM-GCC. It is possible to change the code to work based on other timer or STM F1, F2 or F4 line. This version is using STM HAL library.



**If you like this library, please let me know, follow me @hubmartin.**

**Also subscribing to my electronics youtube channel https://www.youtube.com/user/hubmartin with practical videos and teardowns will motivate me to make more content for community. Thank you.**


Here I explain the using of lib. Under this example below is explained how lib works under the hood

The library is in the /src/ws2812b directory and example of init, redraw and effects are in src/visEffect.c file.

in ws2812b.h you have to set few defines:
```
// GPIO clock peripheral enable command
#define WS2812B_GPIO_CLK_ENABLE() __HAL_RCC_GPIOA_CLK_ENABLE()
// LED output port.
#define WS2812B_PORT GPIOA
// LED output pins
#define WS2812B_PINS (GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3)
// This all indicates that led data pin is on A0 and others on A1, A2 and A3.

// How many LEDs are in the series
#define WS2812B_NUMBER_OF_LEDS 60
// Number of paralel LED strips on the SAME gpio. Each has its own buffer.
#define WS2812_BUFFER_COUNT 2
```

Then in your code initialize the structure and call the init function:
```
// RGB Framebuffers
uint8_t frameBuffer[3*WS2812B_NUMBER_OF_LEDS];
uint8_t frameBuffer2[3*20];

void visInit(char* order)
{
	// Handling of different orders of RGB
	indexR = strchr(order, 'R') - order;
	indexG = strchr(order, 'G') - order;
	indexB = strchr(order, 'B') - order;

	// Set output channel/pin, GPIO_PIN_0 = 0, for GPIO_PIN_5 = 5 - this has to correspond to WS2812B_PINS
	ws2812b.item[0].channel = 0;
	// Your RGB framebuffer
	ws2812b.item[0].frameBufferPointer = frameBuffer;
	// RAW size of framebuffer
	ws2812b.item[0].frameBufferSize = sizeof(frameBuffer);

	// If you need more parallel LED strips, increase the WS2812_BUFFER_COUNT value
	ws2812b.item[1].channel = 3;
	ws2812b.item[1].frameBufferPointer = frameBuffer2;
	ws2812b.item[1].frameBufferSize = sizeof(frameBuffer2);

	ws2812b_init();
}
```
Order of RGB is often GRB, so if you don't know your strips configuration it's recommended you try that first.

You can also use one framebuffer on many outputs.

When the framebuffer is shorter than the WS2812B_NUMBER_OF_LEDS the framebuffer wraps over, nothing breaks. This is great if you would like to have 500 LEDs in one strip but you only need to repeat 8,16,.. animated pixels.

You can also have one big framebuffer and point the "frameBufferPointer" to different places in your buffer.


### Then you just call the ws2812b_handle() function. You have to trigger new transfer by setting the ws2812b.startTransfer = 1;
```
void visHandle()
{

	if(ws2812b.transferComplete)
	{
		// Update your framebuffer here or swap buffers
		visHandle2();

		// Signal that buffer is changed and transfer new data
		ws2812b.startTransfer = 1;
		ws2812b_handle();
	}
}
```

##How the lib works

WS2812B has specific communication protocol and it is necessary to bend some standard peripherals to use this protocol. The most easy approach would be implement blocking cycle-exact code in assembly that will spit out ones and zeroes as we need. This is implemented in NeoPixel library for AVR and ARM processor.

Some improvements can be made if part of the bits are generated by some HW peripheral for example SPI or PWM. The best solution is to keep the CPU free and let the DMA and timer do the precise timing. So I've made some DMA tests and it appeared that I can use my code to precise generate necessary waveforms. The I looked around and discovered that this is the approach that already OctoWS2811 library is using. So at least I wasn't completely wrong. But I would like to have better lib. I have documented all approaches on my site http://www.martinhubacek.cz/arm/improved-stm32-ws2812b-library.

The octo lib is great when you use lot of parallel strips. Because it can output data for all your 16 strips. On the other side if you use only 2 or 4 LED strips (outputs) then you are wasting your RAM because of the data organization in the library.

Based on your longest LED strip the octo library creates an array uint16_r buffer[NUMBER_OF_LEDS × 24]. So if you use one strip with ten LEDs then it creates the uint16_t array of length 240 (480 bytes) where only one bit of each item in array is used. So you are wasting 93% of your array in RAM. If you have LED strip connected to the pin PB3 then the 3rd bit of each item in array is used. This is because DMA needs pre-processed data in specific format. I can do better.

The DMA has of course some small jitter but when you run this lib on anything above 40MHz it is fine. The jitter is within the WS2812 specification so don't worry. When I run similar code on old STM32F100 on maximal CPU freq 24MHz (F100 line is slow), I had big jitter when the DMA was battling with CPU interrupt bus access. I overclocked the STM32F100 little bit (exactly twice the speed to 48MHz :Đ) and it works without issues.

##My library has few improvements:

### One separate buffer for your (big) framebuffer and second small internal bitbuffer for DMA.
**RGB framebuffer** - this is one dimension array with {R1, G1, B1, R2, G2, B2, ...} format.

**Bitbuffer** - this is basicaly the same format buffer like the Octo2811 lib uses but it is allocated only for 2 LEDs (2 LEDs on each of 16 output channels)

**Here is the improvement.** I fill the **bitbuffer** on-the-fly in the double-buffering fashion based on DMA_HALF_TRANSFER and DMA_COMPLETE_TRANSFER interrupts. While the data to the first LED is fed, I prepare the next 24 bits in the second part of the bitbuffer for the second LED in the DMA Irq handler in the background. This IRQ bit-juggling was optimized so it's just a small overhead - I'll explain that down below. And while the data from the second part of bitbuffer is send to the second LED, the DMA Half transfer interrupt is fired and in it are prepared another 24 bit data for first LED.

### Bit-banding for bit-juggling in the IRQ
Interrupts has to be very fast. Thats why I tried many ways to "serialize" the bits from framebuffer to bitbuffer. The best solution is to use bit-banding (don't confuse with bit-banging). This is HW accelerated access to single bits in RAM and I've made a practical video some time ago https://www.youtube.com/watch?v=h78DyF1NOio

Because the bits are computed on the background during the transmittion, there is little CPU overhead. For one LED strip it is on 64MHz STM32F3 just about 8% **only during** transmission of the data. The overhead is bigger as you use more parallel LED strips on the same port. It can be like 10% for two LEDs but it don't rise linearly. You have to take into account overhead of IRQ service routine, clearing IRQ bits. The bit-juggling with bit-banding implementation is quite efficient.

Bit-banding can also improve the speed of set pixel function in the OctoWS2811 library. If you try that let me know.

**Comparison of different methods generating WS2812B waveforms is also on my site**
http://www.martinhubacek.cz/arm/improved-stm32-ws2812b-library

On the image below is in yellow waveform for WS2812B LED strip. Blue trace displays in HIGH state when the CPU is doing bit-juggling in the DMA IRQ. The first blue peak is DMA half transfer IRQ, the second peak is DMA complete transfer IRQ. On my board I don't have external crystal so instead of 72MHz I run only 64Mhz. The CPU overhead during transfer is 11% and the IRQ routine takes 3.6 microseconds.

![alt tag](https://github.com/hubmartin/ws2812b_stm32F3/blob/master/WS2812%20scope%20waveform.png)

## Pros and Cons
Pros are when you use less paralel strips on the same GPIO port. You can efficiently use your RAM. But when you use more and more LED strips on the same port, the background overhead of bit-juggling takes more time and the CPU will be more busy. This applies only when you are sending data. If your update rate is 60FPS and you have plenty time between frames - you can do your CPU intensive computation between the LED transfers.

## Porting
To port this code to a different STM32 MCU, you have to check or correct the correct timer, DMA channels and streams. You can find some info here
https://github.com/hubmartin/WS2812B_STM32F3/issues/1
