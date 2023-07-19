# sh1106(oled)&dht11 on stm32

## 腳位設定
1. timer >> 分頻成1Mhz
2. 啟用i2c 
3. 啟用rcc外部晶振

詳見.ioc

## u8g2

### 安裝步驟
1. 在資料夾底下創建一個資料夾(Lib_u8g2)
2. 下載u8g2 >> https://github.com/olikraus/u8g2
3. 複製**csrc**底下的所有檔案到剛剛創建的資料夾中(可以把一些不必要的檔案刪除以及用不到的函式註解掉)
4. 打開keil，對專案資料夾按右鍵 >> option of target 'xxx' >> c/c++ >> include path >> 把剛剛建的資料夾路徑複製進去
5. 對專案資料夾按右鍵 >> manage project item >> 在groups底下新增一個group >> add files >> 把剛剛建的資料夾裡面的所有檔案複製進來
6. 在Appliction/User/Core底下新增'oled.h'、'oled.c'這兩個檔案 >> 最下面有

### 使用方法
### api reference
https://github.com/olikraus/u8g2/wiki/u8g2reference
### all font
https://github.com/olikraus/u8g2/wiki/fntlistall

宣告變數
```c
#include "oled.h"
#include "u8g2.h"
u8g2_t u8g2;
```
初始化
```c
u8g2Init(&u8g2);
```
清除buffer
```c
u8g2_ClearBuffer(&u8g2);
```
清除畫面
```c
u8g2_ClearDisplay(&u8g2);
```
設定字型
```c
u8g2_SetFont(&u8g2,u8g2_font_DigitalDiscoThin_tf);
```
繪製文字
```c
u8g2_DrawStr(&u8g2, 0, 20,"Waiting");
u8g2_DrawStr(&u8g2, 0, 40, strHumi);
```
發送buffer(更新畫面)
```c
u8g2_SendBuffer(&u8g2);
```
格式化字串
snprint >> 確保不會超出記憶體空間
```c
#include "stdio.h"
char strHumi[32];
snprintf(strHumi, 32, "Humi:%.1f %%", Humi);
```

## dht11
### define
```c
#define DHT11_PORT GPIOF
#define DHT11_PIN GPIO_PIN_12
```
### us delay
```c
void delayMicroSecond(uint32_t time){
	__HAL_TIM_SET_COUNTER(&htim2, 0);//set counter 0
	__HAL_TIM_ENABLE(&htim2);//start counter 
	while(__HAL_TIM_GET_COUNTER(&htim2) < time){
	}
	__HAL_TIM_DISABLE(&htim2);//close counter
}
```
### setPin
```c
void setPinOutput(GPIO_TypeDef *GPIOx, uint16_t GPIO_PIN){
	GPIO_InitTypeDef GPIO_InitStruct;
	GPIO_InitStruct.Pin = GPIO_PIN;
	GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;
	GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
	HAL_GPIO_Init(GPIOx, &GPIO_InitStruct);
}

void setPinInput(GPIO_TypeDef *GPIOx, uint16_t GPIO_PIN){
	GPIO_InitTypeDef GPIO_InitStruct;
	GPIO_InitStruct.Pin = GPIO_PIN;
	GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
	GPIO_InitStruct.Pull = GPIO_PULLUP;
	HAL_GPIO_Init(GPIOx, &GPIO_InitStruct);
}
```
### reset dht11
```c
uint8_t dht11RstCheck(){
	uint8_t timer = 0;
	setPinOutput(DHT11_PORT, DHT11_PIN);
	HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, GPIO_PIN_RESET);
	delayMicroSecond(20000);
	HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, GPIO_PIN_SET);
	delayMicroSecond(30);
	setPinInput(DHT11_PORT, DHT11_PIN);
	
	while(!HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN)){
		timer ++;
		delayMicroSecond(1);
	}
	if(timer > 100 || timer < 20){
		return 0;
	}
	
	timer = 0;
	while(HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN)){
		timer ++;
		delayMicroSecond(1);
	}
	if(timer >100 || timer < 20){
		return 0;
	}
	
	return 1;
}
```
### read byte from dht11
```c
uint8_t DHT11ReadByte(){
	uint8_t byte;
	for(int i = 0; i < 8;i ++){
		while(HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN));
		while(!HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN));
		
		delayMicroSecond(40);
		byte <<= 1;
		if(HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN)){
			byte |= 0x01;
		}
	}
	return byte;
}
```
### get Humi and Temp from dht11
```c
uint8_t DHT11GetData(float *Humi, float *Temp){
	int8_t sta = 0;
	uint8_t buffer[5];
	if(dht11RstCheck()){
		for(int i = 0; i< 5 ;i++){
			buffer[i] = DHT11ReadByte();
		}
		if(buffer[0] + buffer[1] + buffer[2] + buffer[3] == buffer[4]){
			uint8_t humiInt = buffer[0];
			uint8_t humiFloat = buffer[1];
			uint8_t tempInt = buffer[2];
			uint8_t tempFloat = buffer[3];
			char tmpHumi[8];
			char tmpTemp[8];
			snprintf(tmpHumi, 8 , "%d.%d",humiInt, humiFloat);
			snprintf(tmpTemp, 8, "%d.%d", tempInt, tempFloat);
			sscanf(tmpHumi, "%f", Humi);
			sscanf(tmpTemp, "%f", Temp);
		}
		sta = 0;
	}
	else
	{
		*Humi = 99;
		*Temp = 99;
		sta = -1;
	}
	return sta;
}
```

### 使用方法
**取樣時間務必要 >= 3s**
```c
float Temp = 0;
float Humi = 0;
DHT11GetData(&Humi, &Temp);
```

## oled.h
```c
#ifndef __oled_H
#define __oled_H
#ifdef __cplusplus
 extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "u8g2.h"
/* USER CODE BEGIN Includes */

/* USER CODE END Includes */



/* USER CODE BEGIN Private defines */

/* USER CODE END Private defines */
#define u8         unsigned char  // ?unsigned char ????
#define MAX_LEN    128  //
#define OLED_ADDRESS  0x78 // oled??????
#define OLED_CMD   0x00  // ???
#define OLED_DATA  0x40  // ???

/* USER CODE BEGIN Prototypes */
 uint8_t u8x8_byte_hw_i2c(u8x8_t *u8x8, uint8_t msg, uint8_t arg_int, void *arg_ptr);
 uint8_t u8x8_gpio_and_delay(u8x8_t *u8x8, uint8_t msg, uint8_t arg_int, void *arg_ptr);
 void u8g2Init(u8g2_t *u8g2);
 #ifdef __cplusplus
}
#endif
#endif /*__ i2c_H */
/* USER CODE END Prototypes */

```

## oled.c
```c
#include "oled.h"
#include "i2c.h"

uint8_t u8x8_byte_hw_i2c(u8x8_t *u8x8, uint8_t msg, uint8_t arg_int, void *arg_ptr)
{
    /* u8g2/u8x8 will never send more than 32 bytes between START_TRANSFER and END_TRANSFER */
    static uint8_t buffer[128];
    static uint8_t buf_idx;
    uint8_t *data;

    switch (msg)
    {
    case U8X8_MSG_BYTE_INIT:
    {
        /* add your custom code to init i2c subsystem */
        MX_I2C1_Init(); //I2C???
    }
    break;

    case U8X8_MSG_BYTE_START_TRANSFER:
    {
        buf_idx = 0;
    }
    break;

    case U8X8_MSG_BYTE_SEND:
    {
        data = (uint8_t *)arg_ptr;

        while (arg_int > 0)
        {
            buffer[buf_idx++] = *data;
            data++;
            arg_int--;
        }
    }
    break;

    case U8X8_MSG_BYTE_END_TRANSFER:
    {
        if (HAL_I2C_Master_Transmit(&hi2c1, (OLED_ADDRESS), buffer, buf_idx, 1000) != HAL_OK)
            return 0;
    }
    break;

    case U8X8_MSG_BYTE_SET_DC:
        break;

    default:
        return 0;
    }

    return 1;
}

void delay_us(uint32_t time)
{
    uint32_t i = 8 * time;
    while (i--)
        ;
}

uint8_t u8x8_gpio_and_delay(u8x8_t *u8x8, uint8_t msg, uint8_t arg_int, void *arg_ptr)
{
    switch (msg)
    {
    case U8X8_MSG_DELAY_100NANO: // delay arg_int * 100 nano seconds
        __NOP();
        break;
    case U8X8_MSG_DELAY_10MICRO: // delay arg_int * 10 micro seconds
        for (uint16_t n = 0; n < 320; n++)
        {
            __NOP();
        }
        break;
    case U8X8_MSG_DELAY_MILLI: // delay arg_int * 1 milli second
        delay_us(1);
        break;
    case U8X8_MSG_DELAY_I2C: // arg_int is the I2C speed in 100KHz, e.g. 4 = 400 KHz
        delay_us(5);
        break;                    // arg_int=1: delay by 5us, arg_int = 4: delay by 1.25us
    case U8X8_MSG_GPIO_I2C_CLOCK: // arg_int=0: Output low at I2C clock pin
        break;                    // arg_int=1: Input dir with pullup high for I2C clock pin
    case U8X8_MSG_GPIO_I2C_DATA:  // arg_int=0: Output low at I2C data pin
        break;                    // arg_int=1: Input dir with pullup high for I2C data pin
    case U8X8_MSG_GPIO_MENU_SELECT:
        u8x8_SetGPIOResult(u8x8, /* get menu select pin state */ 0);
        break;
    case U8X8_MSG_GPIO_MENU_NEXT:
        u8x8_SetGPIOResult(u8x8, /* get menu next pin state */ 0);
        break;
    case U8X8_MSG_GPIO_MENU_PREV:
        u8x8_SetGPIOResult(u8x8, /* get menu prev pin state */ 0);
        break;
    case U8X8_MSG_GPIO_MENU_HOME:
        u8x8_SetGPIOResult(u8x8, /* get menu home pin state */ 0);
        break;
    default:
        u8x8_SetGPIOResult(u8x8, 1); // default return value
        break;
    }
    return 1;
}
void u8g2Init(u8g2_t *u8g2)
{
	u8g2_Setup_sh1106_i2c_128x64_noname_f(u8g2, U8G2_R0, u8x8_byte_hw_i2c, u8x8_gpio_and_delay); 
	u8g2_InitDisplay(u8g2);       
	u8g2_SetPowerSave(u8g2, 0);     
	u8g2_ClearBuffer(u8g2);
}

```
