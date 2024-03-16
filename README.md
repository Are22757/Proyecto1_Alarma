//*********************************************************************************************************************************************************
// Universidad del Valle de Guatemala
// IE2023 Programación de Microcontroladores
// Autor: Lisbeth Ivelisse Arévalo Girón
// Proyecto: Reloj_hora_fecha
// Descripción: Código de proyecto 1
// Hardware: ATMega328P

//******************************************************************************************************************************************************************
//Encabezado
//******************************************************************************************************************************************************************
.include "M328PDEF.inc"

.def uhora = R17
.def dhora = R18
.def umin = R19
.def dmin = R20
.def estado = R21
.def udia= R22
.def ddia= R23
.def umes= R24
.def dmes = R25
.cseg 

.org 0x00
	JMP MAIN		//Vector Reset
.org 0x008			//Vector de ISR: PCINT1
	JMP ISR_PCINT1 //Interrupción PC1 para PC1,PC2, PC3, PC4, PC5
.org 0x001A
	JMP INT_TIMER1  //interrupccion de timer 1

MAIN:
//******************************************************************************************************************************************************************
// Stack Pointer
//******************************************************************************************************************************************************************
	LDI R16, LOW (RAMEND)
	OUT SPL, R16
	LDI R17, HIGH (RAMEND)
	OUT SPH, R17
//******************************************************************************************************************************************************************
//Configuración
//******************************************************************************************************************************************************************
SETUP: 
//Configuración del Timer

	LDI R16, 0b1000_0000 ;Configurar el timer a 8MHz
	LDI R16, (1 << CLKPCE)
	STS CLKPR, R16

	LDI R16, 0b0000_0001
	STS CLKPR, R16

//Configuración de botones
	LDI R16, 0b0011_1110		;Habilitando de PC1 a PC5 como entradas
	OUT PORTC, R16

//Configuración de los LEDS
	LDI R16, 0b0011_0000		;Habilitando PB4 y PB5 como salida
	OUT DDRB, R16

	LDI R16, 0b0000_0001		;Habilitando PC0 como salida
	OUT DDRC, R16

//Configuración displays
	LDI R16, 0b1111_1110		;Habilitando de PD1 a PD7 como salidas
	OUT DDRD, R16

//Configuración de COMS
	LDI R16, 0b0000_1111		;Habilitando de PB0 a PB3 como salidas
	OUT DDRB, R16

//Habilitación de interrupciones PIN CHANGE

	//			 PC5			  PC4			PC3            PC2			PC1
	LDI R16,(1 << PCINT13)| (1 << PCINT12)|(1 << PCINT11)|(1 << PCINT10)|(1 << PCINT9)
	STS PCMSK1, R16		;Habilitar PCINT en los pines PCINT10, PCINT11 y PCINT12 (la máscara PCMSK1 hace referencia al puerto C)

	LDI R16, (1 << PCIE1)
	STS PCICR, R16		;Habilitando la ISR PCINT1 [14:8]

	SEI					;Habilitamos Interrupciones Globales GIE

	CALL INT_T1			//llamamos la inicializacion del timer1

	CLR umin
	CLR dmin
	CLR uhora 
	CLR dhora 
	CLR estado
	CLR udia
	CLR ddia
	CLR umes 
	CLR dmes

	LDI uhora, 3
	LDI dhora, 2
	LDI umin, 0
	LDI dmin, 5

	LDI umes, 1
	LDI dmes, 0
	LDI udia, 1
	LDI ddia, 0

	LDI estado, 0

//******************************************************************************************************************************************************************
//Configuración LOOP
//******************************************************************************************************************************************************************

LOOP:	
	CPI estado, 0
	BREQ pre_estado0

	CPI estado, 1
	BREQ pre_estado1

	CPI estado, 2
	BREQ pre_estado2

	CPI estado, 3
	BREQ pre_estado3

	JMP LOOP

//Redireccionamiento de estados
	pre_estado0:
	JMP estado0

	pre_estado1:
	JMP estado1

	pre_estado2:
	JMP estado2

	pre_estado3:
	JMP estado3

	

	
//******************************************************************************************************************************************************************
//Estados
//******************************************************************************************************************************************************************

estado0:
	SBI PINC, 0

	//Mostrar valores decenas de horas
	LDI R16, 0b0000_1110  //Multiplexación COMS
	OUT PORTB, R16


	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, dhora

	LPM R16, Z
	OUT PORTD, R16
	 

	CALL DELAY
	
	//Mostrar valores unidades de horas
	LDI R16, 0b0000_1101
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1) 
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, uhora
	LPM R16, Z
	OUT PORTD, R16 

	CALL DELAY

	//Mostrar valores decenas de minutos
	LDI R16, 0b0000_1011
	OUT PORTB, R16
	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, dmin
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY


	//Mostrar valores unidades de minutos
	LDI R16, 0b0000_0111
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, umin
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY

//******************************************************************************************************************************************************************
//OVERFLOWS
//******************************************************************************************************************************************************************
	CPI R29, 60	//R29 es el contador relacionado
	BREQ AUMIN1	//Aumento unidades de minutos	

	CPI umin, 10
	BREQ AUMIN2 //Aumento decenas de minutos

	CPI dmin, 6
	BREQ AUHORA1 //Aumento unidades de hora

	CPI uhora, 10
	BREQ AUHORA2 //Aumento decenas de hora

	CPI dhora, 2
	BREQ REST

	JMP LOOP
//******************************************************************************************************************************************************************
//CONFIGURACION DE LOS OVERFLOWS
//******************************************************************************************************************************************************************
AUMIN1:
	LDI R29, 0
	INC umin
	JMP estado0
AUMIN2:                
	LDI umin, 0
	INC dmin
	JMP estado0
AUHORA1: 				
	LDI dmin, 0
	INC uhora
	JMP estado0
AUHORA2:				
	LDI uhora, 0
	INC dhora
	JMP estado0
REST:				
	CPI uhora, 4
	BREQ RESET
	JMP LOOP
RESET: 
	LDI umin, 0
	LDI dmin, 0
	LDI uhora, 0
	LDI dhora, 0
	INC udia
	JMP estado0
//******************************************************************************************************************************************************************
//Estado1_Demostración hora
//******************************************************************************************************************************************************************


estado1:
	CBI PINC, 0
	// Mostrar valores decenas día
	LDI R16, 0b0000_1110  //Multiplexación COMS
	OUT PORTB, R16


	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, ddia

	LPM R16, Z
	OUT PORTD, R16
	 

	CALL DELAY
	

	//Mostrar valores unidades día
	LDI R16, 0b0000_1101
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1) 
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, udia
	LPM R16, Z
	OUT PORTD, R16 

	CALL DELAY

	//Mostrar valores decenas mes
	LDI R16, 0b0000_1011
	OUT PORTB, R16
	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, dmes
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY

	//Mostrar valores unidades mes
	LDI R16, 0b0000_0111
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, umes
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY
//******************************************************************************************************************************************************************
//OVERFLOWS
//******************************************************************************************************************************************************************

	CPI udia, 10
	BREQ DIA1
	CPI ddia, 3
	BREQ DIA11
	CPI umes, 10
	BREQ MES1
	CPI dmes, 1
	BREQ MES11


	JMP LOOP
//******************************************************************************************************************************************************************
//CONFIGURACION DE LOS OVERFLOWS
//******************************************************************************************************************************************************************

DIA1:               
	LDI udia, 0						//Acá se indica que al transcurrir 10 días se cambia el valor de las decenas de día
	INC ddia
	JMP estado1

DIA11:							//Acá se indica que cada 31 días se cambia el valor para poner nuevamente los días a 01
	CPI udia, 2
	BREQ OVER
	JMP LOOP
OVER: 
	LDI udia, 1
	LDI ddia, 0
	JMP estado1

MES1:                
	LDI umes, 0					//MES1 cambiará cada 10 meses el valor para poner las decenas del mes.
	INC dmes
	JMP estado1

MES11:							//MES11 cambiará el valor para poner los meses a 01.
	CPI umes, 3
	BREQ OVERM
	JMP LOOP

OVERM: 
	LDI umes, 1
	LDI dmes, 0
	JMP estado1
//******************************************************************************************************************************************************************
//Estado2_Demostración fecha
//******************************************************************************************************************************************************************

estado2:
	SBI PINC, 0

	LDI R16, 0b0000_1110  //Multiplexación COMS
	OUT PORTB, R16

	//Mostrar valores de las decenas de las horas
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, dhora

	LPM R16, Z
	OUT PORTD, R16
	 

	CALL DELAY
	//Mostrar valores de las unidades de las horas
	LDI R16, 0b0000_1101
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1) 
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, uhora
	LPM R16, Z
	OUT PORTD, R16 

	CALL DELAY

	//Mostrar valores de las decenas de los minutos
	LDI R16, 0b0000_1011
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, dmin
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY


	//Mostrar valores de las unidades de los minutos
	LDI R16, 0b0000_0111
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, umin
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY

//******************************************************************************************************************************************************************
//OVERFLOWS
//******************************************************************************************************************************************************************

//Comparaciones para redireccionamiento a overflows de hora
	CPI R29, 60
	BREQ AUMIN1_CONFI		
	CPI umin, 10
	BREQ AUMIN2_CONFI
	CPI dmin, 6
	BREQ AUHORA1_CONFI
	CPI uhora, 10
	BREQ AUHORA2_CONFI
	CPI dhora, 2
	BREQ REST_CONFI

	JMP LOOP
//******************************************************************************************************************************************************************
//CONFIGURACION DE LOS OVERFLOWS
//******************************************************************************************************************************************************************
AUMIN1_CONFI:					//Aumento unidades de minuto en base al estado del contador R29
	LDI R29, 0
	INC umin
	JMP estado2
AUMIN2_CONFI:					//Aumento decenas de minuto
	LDI umin, 0
	INC dmin
	JMP estado2
AUHORA1_CONFI: 					//Aumento unidades de hora
	LDI dmin, 0
	JMP estado2
AUHORA2_CONFI:					//Aumento decenas de hora
	LDI uhora, 0
	INC dhora
	JMP estado2
REST_CONFI:				
	CPI uhora, 4
	BREQ RESET_CONFI
	JMP LOOP
RESET_CONFI: 
	LDI uhora, 0
	LDI dhora, 0
	JMP estado2

//******************************************************************************************************************************************************************
//Estado3_Configuración hora
//******************************************************************************************************************************************************************	

estado3:
	CBI PINC, 0

	//Mostrar valores decenas día
	LDI R16, 0b0000_1110  //Multiplexación COMS
	OUT PORTB, R16


	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, ddia

	LPM R16, Z
	OUT PORTD, R16
	 

	CALL DELAY
	
	//Mostrar valores unidades día
	LDI R16, 0b0000_1101
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1) 
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, udia
	LPM R16, Z
	OUT PORTD, R16 

	CALL DELAY

	//Mostrar valores decenas mes
	LDI R16, 0b0000_1011
	OUT PORTB, R16
	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, dmes
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY

	//Mostrar valores unidades mes
	LDI R16, 0b0000_0111
	OUT PORTB, R16

	
	LDI ZH, HIGH(TABLA7SEG << 1)  
	LDI ZL, LOW(TABLA7SEG << 1)
	ADD ZL, umes
	LPM R16, Z
	OUT PORTD, R16
	
	CALL DELAY

//******************************************************************************************************************************************************************
//OVERFLOWS
//******************************************************************************************************************************************************************
	
	//Redireccionamiento de fecha para realizar overflows
	CPI udia, 10
	BREQ DIA1_CONFI

	CPI ddia, 3
	BREQ DIA11_CONFI

	CPI umes, 10
	BREQ MES1_CONFI

	CPI dmes, 1
	BREQ MES11_CONFI

	JMP LOOP

//******************************************************************************************************************************************************************
//CONFIGURACION DE LOS OVERFLOWS
//******************************************************************************************************************************************************************	
DIA1_CONFI:               
	LDI udia, 0							//CADA 10 DIAS SE CAMBIA EL VALOR PARA PONER LAS DECENAS DEL DIA
	INC ddia
	JMP estado3

DIA11_CONFI:							//Cada 31 días se cambia el valor para porner los días a 01. 
	CPI udia, 2
	BREQ OVER_CONFI
	JMP LOOP
OVER_CONFI:								// Se configura que la fecha indique el respectivo 01 respecto al día.
	LDI udia, 1
	LDI ddia, 0
	JMP estado3

MES1_CONFI:                
	LDI umes, 0							//Cada 10 meses se cambia el valor para poner las decenas del mes.
	INC dmes
	JMP estado3
MES11_CONFI:							//Cada 12 meses se cambia el valor para poner los meses a 01.
	CPI umes, 3
	BREQ OVERM_CONFI
	JMP LOOP
OVERM_CONFI:							// Se configura que la fecha indique el respectivo 01 respecto al mes.
	LDI umes, 1
	LDI dmes, 0
	JMP estado3

//******************************************************************************************************************************************************************
// Subrutina de ISR PCINT1
//******************************************************************************************************************************************************************
ISR_PCINT1:
	PUSH R16  ;Guardamos en la pila el registro R16
	IN  R16, SREG
	PUSH R16  ;Guardamos en la pila el registro SREG
	
	IN R16, PINC

	SBRC R16, PC1
	JMP comparar_estados  //Hará un salto a la rutina que compara el valor actual del contador estado
	
	INC estado
	CPI estado, 4
	BRNE SALIR
	LDI estado, 0
	JMP SALIR

comparar_estados:		//Redirecciona a la subrutina respectiva a la comparación del valor de "estado"
	CPI estado, 2		// estado = 2?
	BREQ PC_estado2

	CPI estado,3		// estado = 3?
	BREQ PC_estado3

	JMP SALIR

PC_estado2:					// Referente al estado configuración de hora
	SBRC R16, 2				// PC2 incrementa las unidades de los minutos
	INC umin	
	SBRC R16, 3				// PC3 decrementa las unidades de los minutos
	DEC umin	

	CPI umin, -1			//underflow de las unidades de los minutos
	BRNE BOTONES			
	LDI umin, 9
	DEC dmin
	CPI dmin, -1			//underflow de las decenas de los minutos
	BRNE BOTONES
	LDI dmin, 5

BOTONES:
	SBRC R16, 4				// PC4 incrementa las unidades de las horas (Si llega a su máximo valor se genera el overflow)
	INC uhora	
	SBRC R16, 5				// PC5 decrementa las unidades de las horas (Si llega a su mínimo valor se genera el underflow)
	DEC uhora		

	CPI uhora, -1			//underflow de las unidades de hora
	BRNE SALIR
	LDI uhora, 9				
	DEC dhora
	CPI dhora, -1			//underflow de las decenas de hora
	BRNE SALIR
	LDI dhora, 2
	LDI uhora, 3 
	 
	JMP SALIR

PC_estado3:                 // Referente al estado configuración de fecha
	SBRC R16, 2				// PC2 incrementa las unidades de los días (Si llega a su máximo valor se genera el overflow)
	INC udia	
	SBRC R16, 3				// PC3 decrementa las unidades de los días (Si llega a su mínimo valor se genera el underflow)
	DEC udia	

	CPI udia, -1			//comparo para redireccionar a la rutina BOTONES_FECHA (underflow de las unidades de los días)
	BRNE BOTONES_FECHA
	LDI udia, 9
	DEC ddia
	CPI ddia, -1			//comparo para redireccionar a la rutina BOTONES_FECHA (underflow de las decenas de los días)
	BRNE BOTONES_FECHA
	LDI ddia, 3
	LDI udia, 1  
	JMP BOTONES_FECHA

BOTONES_FECHA:
	SBRC R16, 4				// PC4 incrementa las unidades de los meses (Si llega a su máximo valor se genera el overflow)
	INC umes	
	SBRC R16, 5				// PC5 decrementa las unidades de los meses (Si llega a su mínimo valor se genera el underflow)
	DEC umes
	
	CPI umes, -1			//underflow de las unidades de los meses
	BRNE SALIR
	LDI umes, 9
	DEC dmes
	CPI dmes, -1			//underflow de las decenas de los meses
	BRNE SALIR


	LDI dmes, 1
	LDI umes, 2  
	JMP SALIR	

SALIR:
	POP R16
	OUT SREG, R16
	POP R16
	RETI

//******************************************************************************************************************************************************************
//Subrutina para multiplexación
//******************************************************************************************************************************************************************
DELAY: ;Necesaria para realizar la multiplexación
	LDI R16,255
DELAY1:
	DEC R16
	BRNE DELAY1
	LDI R16, 255
DELAY2:
	DEC R16
	BRNE DELAY2
	LDI R16, 255
DELAY3:
	DEC R16
	BRNE DELAY3
	LDI R16, 255
DELAY4:
	DEC R16
	BRNE DELAY4

	RET

//******************************************************************************************************************************************************************
//Subrutina Timer1
//******************************************************************************************************************************************************************

INT_T1:
	LDI R16, 0
	STS TCCR1A, R16   //inicializacion de timer 1 como contador 
	
	LDI R16, (1<<CS02) | (1<<CS00)     //seleccion de prescaler de 1024 
	STS TCCR1B, R16       
	
	LDI R16, 0xE1		//valores prescalares para el conteo 
	STS TCNT1H, R16
	LDI R16, 0x7B
	STS TCNT1L, R16

	LDI R16, (1<<TOIE1)   //mascara del timer1
	STS TIMSK1, R16

	RETI

INT_TIMER1:
	PUSH R16			//guardamos el valor de R16
 	IN R16, SREG
	PUSH R16
     
	LDI R16, 0xE1		//valores prescalares para el conteo 
	STS TCNT1H, R16
	LDI R16, 0x7B      
	STS TCNT1L, R16		//Se carga el valor inicial del contador
	SBI TIFR1, TOV1		//Se borra la bandera de TOV0
	
	INC R29				//Se incrementa el contador desginado para el timer

	POP R16
	OUT SREG, R16 
	POP R16				//Devolvemos el valor antes guardado

	RETI				//retorno de interrupcion 

//******************************************************************************************************************************************************************
TABLA7SEG: .DB 0b1111_1100, 0b0010_0100, 0b1011_1010, 0b1010_1110, 0b0110_0110,0b1100_1110,0b1101_1110, 0b1010_0100, 0b1111_1110, 0b1110_0110
