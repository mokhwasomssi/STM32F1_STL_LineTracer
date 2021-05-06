#include <stm32f10x_rcc.h> 
#include <stm32f10x_gpio.h> 
#include <stm32f10x_tim.h>
#include <stm32f10x_adc.h>
#include <stm32f10x_dma.h> 
#include <stm32f10x_usart.h>
#include <stm32f10x_spi.h>
#include <stdio.h> //printf 쓰기위해 필요한 헤더 파일

vu16 ADC_Value[6];

//coretex-m3  stm32F103C8T6

static void delay(volatile unsigned int nTime) //줄처리 소요 시간을
{                                                        //지연시키는 함수
  for(; nTime > 0 ; nTime--);
}
 
void RCC_Configuration(void) //클럭 제공
{                           
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA , ENABLE); // 센서 - PA01~05 , USART PA09~10
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB , ENABLE); //LED
  
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE); 
  
  RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2 , ENABLE); //타이머2

  RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE); //USART
  
  RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);
 
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 , ENABLE);
}

void GPIO_Configuration() // 센서 , USART , LED , 모터 
{
  GPIO_InitTypeDef GPIOA_Sensor; //센서
  GPIOA_Sensor.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 |GPIO_Pin_2 | GPIO_Pin_3 | GPIO_Pin_4 | GPIO_Pin_5 ;
  GPIOA_Sensor.GPIO_Speed = GPIO_Speed_50MHz; 
  GPIOA_Sensor.GPIO_Mode = GPIO_Mode_AIN;
  GPIO_Init(GPIOA, &GPIOA_Sensor);
  
  GPIO_InitTypeDef GPIOA_USART_Tx; //USART_Tx
  GPIOA_USART_Tx.GPIO_Pin = GPIO_Pin_9;
  GPIOA_USART_Tx.GPIO_Speed = GPIO_Speed_50MHz;
  GPIOA_USART_Tx.GPIO_Mode = GPIO_Mode_AF_PP;
  GPIO_Init(GPIOA, &GPIOA_USART_Tx);
  
  GPIO_InitTypeDef GPIOA_USART_Rx;//USART_Rx
  GPIOA_USART_Rx.GPIO_Pin = GPIO_Pin_10;
  GPIOA_USART_Rx.GPIO_Speed = GPIO_Speed_50MHz; 
  GPIOA_USART_Rx.GPIO_Mode = GPIO_Mode_IN_FLOATING;
  GPIO_Init(GPIOA, &GPIOA_USART_Rx);  
  
  //LED
  GPIO_InitTypeDef GPIOB_LED ; 
  GPIOB_LED.GPIO_Pin = GPIO_Pin_3 | GPIO_Pin_4| GPIO_Pin_5 | GPIO_Pin_6 | GPIO_Pin_7 | GPIO_Pin_8 ;
  GPIOB_LED.GPIO_Speed = GPIO_Speed_50MHz; 
  GPIOB_LED.GPIO_Mode =GPIO_Mode_Out_PP; 
  GPIO_Init(GPIOB, &GPIOB_LED);
  
  GPIO_PinRemapConfig(GPIO_Remap_SWJ_Disable, ENABLE); //리맵핑 PB핀 기능 변경
  
  GPIO_InitTypeDef GPIOB_TIM2; //TIM2-Chaneel 3 , 4 
  GPIOB_TIM2.GPIO_Pin = GPIO_Pin_10 | GPIO_Pin_11 ; 
  GPIOB_TIM2.GPIO_Speed = GPIO_Speed_50MHz; 
  GPIOB_TIM2.GPIO_Mode = GPIO_Mode_AF_PP;
  GPIO_Init(GPIOB, &GPIOB_TIM2);
  
  GPIO_PinRemapConfig(GPIO_FullRemap_TIM2, ENABLE);  //PB10 - TIM2_CH3 , PB11 - TIM2_CH4
 
  GPIO_InitTypeDef GPIO_Input; //input1,2,3,4
  GPIO_Input.GPIO_Pin =  GPIO_Pin_12 | GPIO_Pin_13 | GPIO_Pin_14 | GPIO_Pin_15;
  GPIO_Input.GPIO_Speed = GPIO_Speed_50MHz; 
  GPIO_Input.GPIO_Mode = GPIO_Mode_Out_PP; 
  GPIO_Init(GPIOB, &GPIO_Input);
  
 
 }

void TIM_Configuration() //타이머2 설정
{
  uint16_t TimerPeriod = 7199; 
  TIM_TimeBaseInitTypeDef  TIM_TimeBaseStructure; 
  TIM_TimeBaseStructure.TIM_Prescaler = 9999; 
  TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
  TIM_TimeBaseStructure.TIM_Period = TimerPeriod; 
  TIM_TimeBaseStructure.TIM_ClockDivision = 0; 
  TIM_TimeBaseStructure.TIM_RepetitionCounter = 0;  
  TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);
  TIM_Cmd(TIM2, ENABLE);
 }

void PWM_Configuration_Channel3() // 타이머 리매핑 , 모터1 제어 , main 함수에 Tim_Pulse 설정 (속도 제어)
{
    TIM_OCInitTypeDef TIM_OCInitStructure;  
    TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
    TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;  
    TIM_OCInitStructure.TIM_Pulse =0;
    TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_Low;
    TIM_OC3Init(TIM2, &TIM_OCInitStructure); 
    TIM_OC3PreloadConfig(TIM2, TIM_OCPreload_Enable);
 }

void PWM_Configuration_Channel4() //모터2 제어
{
    TIM_OCInitTypeDef TIM_OCInitStructure_Channel4;  
    TIM_OCInitStructure_Channel4.TIM_OCMode = TIM_OCMode_PWM1;
    TIM_OCInitStructure_Channel4.TIM_OutputState = TIM_OutputState_Enable;  
    TIM_OCInitStructure_Channel4.TIM_Pulse = 0;
    TIM_OCInitStructure_Channel4.TIM_OCPolarity = TIM_OCPolarity_Low;
    TIM_OC4Init(TIM2, &TIM_OCInitStructure_Channel4); 
    TIM_OC4PreloadConfig(TIM2, TIM_OCPreload_Enable);
 }

void DMA_Configuration() //ADC1
{
  DMA_InitTypeDef D;
  
  DMA_DeInit(DMA1_Channel1);
  D.DMA_PeripheralBaseAddr = (uint32_t)&ADC1->DR;//((uint32_t)0x4001244C);
  D.DMA_MemoryBaseAddr = (uint32_t)&ADC_Value;
  D.DMA_DIR = DMA_DIR_PeripheralSRC;
  D.DMA_BufferSize = 6;
  D.DMA_PeripheralInc = DMA_PeripheralInc_Disable;
  D.DMA_MemoryInc = DMA_MemoryInc_Enable ;
  D.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;
  D.DMA_MemoryDataSize = DMA_MemoryDataSize_HalfWord;
  D.DMA_Mode = DMA_Mode_Circular;
  D.DMA_Priority =  DMA_Priority_High;
  D.DMA_M2M = DMA_M2M_Disable;
  DMA_Init(DMA1_Channel1 , &D);
  
  DMA_Cmd(DMA1_Channel1 ,  ENABLE);
}

void USART_Configuration() //통신
{
  USART_InitTypeDef U;
  U.USART_BaudRate = 1200;
  U.USART_WordLength = USART_WordLength_8b;
  U.USART_StopBits = USART_StopBits_1;
  U.USART_Parity = USART_Parity_No;
  U.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
  U.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
  
  USART_Init(USART1 , &U );
  
  USART_DMACmd( USART1 , USART_DMAReq_Tx | USART_DMAReq_Rx, ENABLE);
  
  USART_Cmd(USART1, ENABLE);
  
  DMA_Cmd(DMA1_Channel4, ENABLE);
  DMA_Cmd(DMA1_Channel5, ENABLE);
  
  
}
     
 void ADC1_Configuration() //센서
{
  ADC_InitTypeDef ADC1_InitStructure;
    
  ADC1_InitStructure.ADC_Mode = ADC_Mode_Independent; 
  ADC1_InitStructure.ADC_ScanConvMode = ENABLE ; 
  ADC1_InitStructure.ADC_ContinuousConvMode = ENABLE; 
  ADC1_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;
  ADC1_InitStructure.ADC_DataAlign = ADC_DataAlign_Right; 
  ADC1_InitStructure.ADC_NbrOfChannel = 6;
  ADC_Init(ADC1 ,&ADC1_InitStructure);
  
  ADC_Cmd(ADC1, ENABLE);
  
  ADC_ResetCalibration(ADC1);
   
  while(ADC_GetResetCalibrationStatus(ADC1));
    ADC_StartCalibration(ADC1);
  while(ADC_GetCalibrationStatus(ADC1));
    ADC_SoftwareStartConvCmd(ADC1, ENABLE); // Enables the selected ADC software start conversion
   
   //센서 채널 설정 - 6개 
  
  ADC_RegularChannelConfig(ADC1 , ADC_Channel_0 , 1 , ADC_SampleTime_1Cycles5);
  ADC_RegularChannelConfig(ADC1 , ADC_Channel_1 , 2 , ADC_SampleTime_1Cycles5);
  ADC_RegularChannelConfig(ADC1 , ADC_Channel_2 , 3 , ADC_SampleTime_1Cycles5);
  ADC_RegularChannelConfig(ADC1 , ADC_Channel_3 , 4 , ADC_SampleTime_1Cycles5);
  ADC_RegularChannelConfig(ADC1 , ADC_Channel_4 , 5 , ADC_SampleTime_1Cycles5);
  ADC_RegularChannelConfig(ADC1 , ADC_Channel_5 , 6 , ADC_SampleTime_1Cycles5);
  
   
 
  ADC_DMACmd(ADC1, ENABLE); //Enable DMA for ADC
} 


int fputc(int ch , FILE*f) //printf를 쓰기 위해 필요한 함수 (이유는 모름ㅎㅎ)
{
  if(ch=='\n')
  {
  USART_SendData(USART1 , '\r');
  while(USART_GetFlagStatus(USART1 , USART_FLAG_TXE) == RESET);
  USART_SendData(USART1 , '\n');
  }
  else
  {
    USART_SendData(USART1 , (uint8_t) ch);
  }
  
  while(USART_GetFlagStatus(USART1 , USART_FLAG_TXE)==RESET);
  return ch;

}


void Go() { GPIO_SetBits(GPIOB ,GPIO_Pin_12|   GPIO_Pin_14 );  GPIO_ResetBits(GPIOB , GPIO_Pin_13| GPIO_Pin_15); } //

void Left() { GPIO_SetBits(GPIOB ,GPIO_Pin_13|   GPIO_Pin_14 );  GPIO_ResetBits(GPIOB , GPIO_Pin_12| GPIO_Pin_15); }

void Right() { GPIO_SetBits(GPIOB ,GPIO_Pin_12|   GPIO_Pin_15 );  GPIO_ResetBits(GPIOB , GPIO_Pin_13| GPIO_Pin_14); }



void main(void)
{
  RCC_Configuration();
  GPIO_Configuration();
  TIM_Configuration();
  
  PWM_Configuration_Channel3();
  PWM_Configuration_Channel4() ;
  
  DMA_Configuration();
  USART_Configuration();
  ADC1_Configuration();
 
  GPIO_ResetBits(GPIOB , GPIO_Pin_4);
  
  
 
  
 /*GPIO_SetBits(GPIOB , GPIO_Pin_10 | GPIO_Pin_11 );

 GPIO_SetBits(GPIOB ,GPIO_Pin_12|   GPIO_Pin_14 );
  
 GPIO_ResetBits(GPIOB , GPIO_Pin_13| GPIO_Pin_15);*/

 
while(1)
{
 //printf(" %d\t  %d\t  %d\t  %d\t  %d\t  %d\n", ADC_Value[0],ADC_Value[1],ADC_Value[2],ADC_Value[3],ADC_Value[4],ADC_Value[5]); //센서값 받기 
  
  
 //Sensor to LED
 if(ADC_Value[0] >=300) GPIO_SetBits(GPIOB , GPIO_Pin_8); 
 else if(ADC_Value[0]<300)  {GPIO_ResetBits(GPIOB , GPIO_Pin_8); Right(); }
  
 if(ADC_Value[1] >=300) GPIO_SetBits(GPIOB , GPIO_Pin_7);
 else if(ADC_Value[1]<300) {GPIO_ResetBits(GPIOB , GPIO_Pin_7); Right(); }

 if(ADC_Value[2] >=300) GPIO_SetBits(GPIOB , GPIO_Pin_6);
 else if(ADC_Value[2]<300)  {GPIO_ResetBits(GPIOB , GPIO_Pin_6); Go();}

 if(ADC_Value[3] >=300) GPIO_SetBits(GPIOB , GPIO_Pin_5);
 else if(ADC_Value[3]<300) {GPIO_ResetBits(GPIOB , GPIO_Pin_5); Go();}

 if(ADC_Value[4] >=300) GPIO_SetBits(GPIOB , GPIO_Pin_4);
 else if(ADC_Value[4]<300) {GPIO_ResetBits(GPIOB , GPIO_Pin_4); Left();}
 
 if(ADC_Value[5] >=30) GPIO_SetBits(GPIOB , GPIO_Pin_3);
 else if(ADC_Value[5]<300) {GPIO_ResetBits(GPIOB , GPIO_Pin_3); Left(); }
 //Sensor to LED
 
 
 
} 
  
  
  
  
 }