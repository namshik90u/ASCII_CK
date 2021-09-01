[FND_TEST.txt](https://github.com/namshik90u/ASCII_CK/files/7088495/FND_TEST.txt)
[TIMER_c.txt](https://github.com/namshik90u/ASCII_CK/files/7088498/TIMER_c.txt)
[UART_C.txt](https://github.com/namshik90u/ASCII_CK/files/7088501/UART_C.txt)

```#include <avr/io.h>
#include <util/delay.h>
#include <avr/interrupt.h>
#include <string.h>


#define sbi(REG,BIT) REG |= (1 << BIT)
#define cbi(REG,BIT) REG &= ~(1 << BIT)
#define TIMER0_CONSTANT 83;
#define TIMER0_START TCCR0B = 0x03;
#define TIMER0_STOP TCCR0B = 0x00;
#define TIMER_FND_OFF PORTC = 0xff;

unsigned char Timer_FND_Digit[17] = {0xc0, 0xee, 0xa1, 0xa4, 0x8e, 0x94, 0x98, 0xe6,
                                     0x80, 0x86, 0xee, 0xa8, 0xbf, 0x91, 0xba, 0xff, 0x93
                                    };
                                    
volatile unsigned char Timer_FND_COM = 0;
volatile unsigned int Timer0_Tick = 0;

unsigned char Alarm_Hour = 1;
unsigned char Alarm_Min = 15;

volatile unsigned char FND_Value3 = 12;
volatile unsigned char FND_Value2 = 21;
volatile unsigned char FND_Value1 = 0;



volatile unsigned char cnt = 0;
volatile unsigned int Timer_1ms = 0;

#include "Timer.h"
#include "Uart.h"
unsigned int Delay_value;


void SendTime(int hour,int min,int sec,int visible) 
{
	Send_Data(hour/10+'0');
	Send_Data(hour%10+'0');
	Send_Data(':');
	Send_Data(min/10+'0');
	Send_Data(min%10+'0');
	if(visible==3) 									//visible == 3이면 '초'도 표시 
	{
		Send_Data(':');														
		Send_Data(sec/10+'0');
		Send_Data(sec%10+'0');
	}
	Send_Data(0x0d);								//줄바꿈   
}

int get_number(char str[]) {
	int a,b;
	a=str[0]-'0';
	b=str[1]-'0';
	return a*10+b;
}

int main(void) {

	Timer_Setting();
	Uart_Init();
	DDRC = 0x0f;
	DDRJ = 0xff;
	
	DDRA = 0xff;
	PORTA = 0;

	sei();
	TIMER0_START // TCCT0B = 0x03

	while(1) {
		_delay_ms(100);
		if(Timer0_Tick==0) 
		{
			if(FND_Value3==1 && FND_Value2==15) 	//4번 - 1:15이 되면 알람이 1초에 한번씩 출력
			{ 
				Send("Alarm : ");										//Alarm :
				SendTime(FND_Value3,FND_Value2,FND_Value1,3);	//시 분 초 표시  
			}
			else if(!(PINA & 0x04)) 			  			//3번 - 버튼 누르면 현재시각 표시 
				{ 
				Send("Time : ");										//Time :
				SendTime(FND_Value3,FND_Value2,FND_Value1,3);		//시 분 초가 표시됨 
				}
		}

		if(received) 
		{
			if(strcmp(buf,"NZ")==0) 				//7번 - NZ를 입력하면 현재 시각과 알람 시간 전송 
			{
				Send("Now Time : ");										//NOW Time :
				SendTime(FND_Value3,FND_Value2,FND_Value1,3); 				//현재 시 분 초 
				Send("Setted Alarm : ");									//Setted Alarm :
				SendTime(Alarm_Hour,Alarm_Min,0,2);							//알람 시 분 
			} 
				else if(buf[0]=='A') 					//5번 - A--:--Z를 입력하면 알람시간 변경 후 Set Alarm : 전송 
				{
					Alarm_Hour=get_number(&buf[1]);							//입력한 알람의 시를 Alarm_Hour에 저장 
					Alarm_Min=get_number(&buf[4]);							//입력한 알람의 분을 Alarm_Min에 저장 
					Send("Set Alarm	: ");									//Set Alarm :
					SendTime(Alarm_Hour,Alarm_Min,0,2);						//위에서 저장된 Alarm_Hour, Alarm_Min이 표시됨 
				} 
					else if(buf[0]=='T') 				//6번 - T--:--Z를 입력하여 현재 시간 설정 
					{
					FND_Value3=get_number(&buf[1]);							//get_number로 입력한 숫자가 FND_Value3에 저장 
					FND_Value2=get_number(&buf[4]);							//get_number로 입력한 숫자가 FND_Value2에 저장  
					Send("Set Time : ");									//Set Time :
					SendTime(FND_Value3,FND_Value2,FND_Value1,3);			//변경한 현재 시간 출력 시 분 초 
					}
			Clear();
		}

	}
	return 0;
}


<timer_h>
void Timer_Setting(void)
{
	TIMSK0=0x01;
	TCCR0A=0;
	TCNT0=TIMER0_CONSTANT;
	TIMER0_STOP
}
SIGNAL(SIG_OVERFLOW0)
{
	Timer0_Tick++ ;
	
	if(Timer0_Tick > 5)
	{
		Timer0_Tick = 0;
		TIMER_FND_OFF
		cbi(PORTC,Timer_FND_COM);
		if(Timer_FND_COM == 3)
		{
			PORTJ=Timer_FND_Digit[FND_Value2%10];
		}
		else if(Timer_FND_COM == 2)
		{
			PORTJ = Timer_FND_Digit[(FND_Value2%100)/10];
		}
		else if(Timer_FND_COM == 1)
		{
			PORTJ = Timer_FND_Digit[FND_Value3%10];
		}
		else if(Timer_FND_COM == 0)
		{
			PORTJ = Timer_FND_Digit[(FND_Value3%100)/10];
		}
		Timer_FND_COM++;
		if(Timer_FND_COM==4) Timer_FND_COM=0;
		
   }
   Timer_1ms++;                                           
   if(Timer_1ms > 1000)                   
   {
      Timer_1ms = 0;                                  
      FND_Value1++;                                   
      if(FND_Value1 >= 60)                            
      {
         FND_Value1 = 0;                         
         FND_Value2++;                          
         if(FND_Value2 >= 60)                    
         {
            FND_Value2 = 0;
			cnt++;
			if(cnt >= 24 )
			{
				cnt=0;
			}
         }
      }
   }
   TCNT0 = TIMER0_CONSTANT;
}

<UART.H>

void Uart_Init(void)
{
	UBRR1L = 5;		//11.0592MHz - 115200bps
	UBRR1H = 0;		//
	
	UCSR1A = 0x00;
	UCSR1B = 0x98;
	UCSR1C = 0x06;
}

void Send_Data(unsigned char T)		//UART Channel 1 Send
{
	loop_until_bit_is_set(UCSR1A,UDRE1);
	UDR1 = T;
}

char buf[256];
int bufindex=0;
int received=0;
SIGNAL(SIG_USART1_RECV)
{
	char ch= UDR1;
	buf[bufindex]= ch;
	
	buf[bufindex+1]= '\0';
	bufindex++; 
	if(ch=='Z')
		received=1;
}

void Clear()
{
	bufindex=0;
	received=0;
}

void Send(char T[])	
{
	int i;
	for(i=0;T[i]!=0;i++)
		Send_Data(T[i]);
}
```
