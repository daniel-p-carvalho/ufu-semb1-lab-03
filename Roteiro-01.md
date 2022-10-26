# Laboratório 03

## 1 - Objetivos

Continuar o desenvolvimento do *firmware* para piscar o LED integrado ao kit
de desenvolvimento **STM32F411 Blackpill**.

No laboratório de hoje iremos abordar os seguintes temas:

* esquemático do kit **STM32F411 Blackpill**;
* documentação do microcontrolador **STM32F411**;
* organização da memória de microcontroladores ARM Cortex-M4;
* periférico RCC ***Reset and Clock Control***;
* periférico GPIO ***General Purpose Input/Output***;
* utilizar o periférico GPIO para piscar o LED.

## 2 - Pré-requisitos

* Windows Subsystem for Linux 2;
* GCC - GNU C Compiler;
* GCC ARM Toolchain;
* OpenOCD - Open On Chip Debugger;
* Sistema de controle de versões Git, e;
* Microsoft Visual Studio Code.

## 3 - Referências

[1] [STM32F411 Blackpill Schematic](https://cdn-shop.adafruit.com/product-files/4877/4877_schematic-STM32F411CEU6_WeAct_Black_Pill_V2.0.pdf)

[2] [STM32F411xC Datasheet]()

[3] [RM0383 Reference Manual](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)

[4] [Using the GNU Compiler Collection (GCC)]()

## 4 - Diagrama esquemático do kit de desenvolvimento **STM32F411 Blackpill**

Para desenvolver o programa para piscar o LED da **STM32F411 Blackpill**
devemos determinar em qual pino do microcontrolador o LED está conectado e que
tipo de sinal devemos uutilizar para ligar/desligar o LED. De acordo com o
**STM32F411 Blackpill** Schematic [1] o LED está conectado ao pino **PC13** do
microcontrolador, ou seja, pino 13 da porta C, e para ligar este LED devemos
configurar este pino em nível lógico baixo.

## 5 - Documentação do microcontrolador **STM32F411**

## 7 - Mapeamento da memória de microcontroladores ARM Cortex-M4

### 7.1 - Mapeamento da memória de microcontroladores da linha STM32

## 8 - Periféricos do **STM32F411**

### 8.1 - RCC ***Reset and Clock Control***

O **RCC** é o periférico utilizado para a configuração do clock da CPU, dos
barramentos, dos periféricos internos do microcontrolador e genrenciar os 
sinais de **reset** do dispositivo.

A árvore de clock de cada microcontrolador é específica mas, de uma maneira geral
para microcontroladores da ST, o fluxo é o seguinte:

* é configurada uma origem da referência do clock: interna de alta velocidade,
**HSI** (***High Speed Internal***) via um oscilador dentro do microcontrolador,
interna de baixa velocidade, **LSE** (***Low Speed Internal***) também via
oscilador dentro do microcontrolador ou externo, **HSE** 
(***High Speed External***), geralmente via um cristal externo ou clock
**CMOS**;

* essa referência pode ou não passar por um **PLL** (***Phase-Locked Loop***) e
gerar o clock básico do sistema, conhecido como **SYSCLK**;

* o **SYSCLK** pode ser dividido para gerar o clock básico dos barramentos do
**ARM Cortex M** conhecido como **AHB** (***Advanced High Performance Bus***),
utilizado por dispositivos rápidos, e **APB** (***Advanced Peripheral Bus***),
utilizado por dispositivos mais lentos.

Cabe ao fabricante decidir quais periféricos serão ligados ao **APB** e ao
**AHB**, assim como a quantidade desses barramentos no sistema. Normalmente,
dispositivos rápidos como memórias, USB, Ethernet estão ligados ao AHB e
periféricos mais lentos como SPI, I2C, Timers, estão ligados ao **APB**.
Na partida, é comum também que a fonte de clock usada seja o **HSI**. Isso
pode ser alterado depois.

### 8.2 - GPIO ***General Purpose Input/Output***

Cada porta do microcontrolador, é representada por uma letra, **PA**, **PB**,
**PC**, etc., sendo que cada porta pode agrupar até 16 pinos que são indexados
após o nome da porta, de **PA0** a **PA15** para a porta **A**, por exemplo. A
quantidade de portas e de pinos por porta dependerá do chip em uso. As portas
também são vistas como um periférico mapeado em memória, possuindo um endereço
base e um conjunto de registros de configuração. São esses registros que irão
permitir a configuração dos pinos, extremamente flexíveis, podendo suportar
características como:

*	uso de resistores de pull up ou pull down;
*	topologias de portas push pull ou open drain;
* configurações de interrupções;
*	funções analógicas (ADC e DAC);
*	outros periféricos digitais.

## 9 - Desenvolvimento do firmware para piscar o LED

Na aula anterior criamos os arquivos **main.c**, que a *priori* não faz nada,
o arquivo **startup.c**, onde implementamos parcialmente os vetores de
interrupção, a rotina de tratamento do *reset* com a inicialização de variáveis
e  o arquivo **Makefile** onde escrevemos as regras para automatizar o processo
de compilação.

Vamos iniciar as atividades deste laboratório copiando os arquivos desenvolvidos
no laboratório anterior para uma nova pasta.

```console
foo@bar$ cd
foo@bar$ cd semb1-workspace
foo@bar:$ cp -r lab-02 lab-03
foo@bar:$ cd lab-03
```

Por questões de economia de energia a maioria dos periféricos do microcontrolador
vem por padrão desligados após o **reset**. Como vamos utilizar a porta C, é
necessário habilitar o *clock* desta porta. Isto é feito através do periférico
**Reset and Clock Control (RCC)**.

O periférico **GPIOC** está conectado ao barramento **AHB1** e os registradores
utilizados para controlar os periféricos conectados a este barramento são:

* **RCC_AHB1RSTR** - **RCC AHB1 peripheral reset register**; e,
* **RCC_AHB1ENR** - **RCC AHB1 peripheral clock enable register**.

De acordo com a **seção 6.3.9** do **RM0308 Reference Manual** para ligar o
*clock* do **GPIOC** devemos ajustar o bit 2, **GPIOCEN**, do registrador
**RCC_AHB1ENR** para 1.

Na arquitetura **Cortex-M** os periféricos são mapeados em memória, geralmente
representados por um conjunto de registros que possuem um endereço base. Assim,
para ajustar o valor do bit **GPIOCEN** devemos determinar o endereço do
registrador **RCC_AHB1ENR** e ajustar o valor deste registrador de acordo com
desejado. O registrador **RCC_AHB1ENR** possui um offset de **0x30** em relação
ao endereço base do módulo **RCC** que, segundo o mapa de memória do STM32F411,
é **0x4002 3800**.

Com base nas informações acima podemos escrever o código para definir os 
endereços dos registradores e ajustar o valores de acordo com o necessário para
habilitar a porta **GPIOC**.

Arquivo: **main.c**
```c
#include <stdlib.h>

/* AHB1 Base Addresses ******************************************************/

#define STM32_RCC_BASE       0x40023800     /* 0x40023800-0x40023bff: Reset 
                                               and Clock control RCC */

/* Register Offsets *********************************************************/

#define STM32_RCC_AHB1ENR_OFFSET    0x0030  /* AHB1 Peripheral Clock enable
                                               register */

/* Register Addresses *******************************************************/

#define STM32_RCC_AHB1ENR           (STM32_RCC_BASE+STM32_RCC_AHB1ENR_OFFSET)

/* AHB1 Peripheral Clock enable register */

#define RCC_AHB1ENR_GPIOCEN         (1 << 2)  /* Bit 2:  IO port C clock 
                                                 enable */

int main(int argc, char *argv[])
{
  uint32_t reg;

  /* Ponteiros para registradores */

  uint32_t *pRCC_AHB1ENR  = (uint32_t *)STM32_RCC_AHB1ENR;

  /* Habilita clock GPIOC */

  reg  = *pRCC_AHB1ENR;
  reg |= RCC_AHB1ENR_GPIOCEN;
  *pRCC_AHB1ENR = reg;

  while(1);

  /* Nao deveria chegar aqui */

  return EXIT_SUCCESS;
}
```

Para acionarmos o LED, devemos configurar o pino **PC13** para operar como um
pino de saída. Podemos configurar este pino como saída de duas formas distintas

* saída em dreno aberto com capacidade de *pull-up* ou *pull-down*;

* saída em *push-pull* com capacidade de *pull-up* ou *pull-down*.

De acordo com o **STM32F411 Blackpill Schematic** [1] para acionar o LED
podemos configurar o pino **PC13** tanto como saída em ***push-pull*** quanto
como saída em coletor aberto. Os resistores de *pull-up* e *pull-down* 
devem estar desligados. Isto pode ser feito por meio dos registradores do
periférico **GPIOC**.

Segundo as **seções 8.4.1, 8.4.2 e 8.4.4** do **RM0308 Reference Manual** 
devemos realizar as seguintes configurações:

* **GPIOC_MODER**: ajustar os bits **MODER13[1:0]** para o valor 1 para
configurar **PC13** como pino de saída;
* **GPIOC_OTYPER**: ajustar o bit **OT13** em 0 para fazer a saída tipo
*push-pull*;
* **GPIOC_PUPDR** ajustar os bits **PUPDR13[1:0]** em 0 para desconectar os
resistores de *pull-up* e *pull-down*.

O código abaixo mostra como fazer estas configurações.

Arquivo: **main.c**
```c
#include <stdlib.h>

/* AHB1 Base Addresses ******************************************************/

#define STM32_RCC_BASE       0x40023800     /* 0x40023800-0x40023bff: Reset 
                                               and Clock control RCC */

/* AHB2 Base Addresses ******************************************************/

#define STM32_GPIOC_BASE     0x48000800U    /* 0x48000800-0x48000bff: GPIO 
                                               Port C */

/* Register Offsets *********************************************************/

#define STM32_RCC_AHB1ENR_OFFSET  0x0030   /* AHB1 Peripheral Clock enable
                                              register */

#define STM32_GPIO_MODER_OFFSET   0x0000  /* GPIO port mode register */
#define STM32_GPIO_OTYPER_OFFSET  0x0004  /* GPIO port output type register */
#define STM32_GPIO_PUPDR_OFFSET   0x000c  /* GPIO port pull-up/pull-down
                                             register */

/* Register Addresses *******************************************************/

#define STM32_RCC_AHB1ENR        (STM32_RCC_BASE+STM32_RCC_AHB1ENR_OFFSET)

#define STM32_GPIOC_MODER        (STM32_GPIOC_BASE+STM32_GPIO_MODER_OFFSET)
#define STM32_GPIOC_OTYPER       (STM32_GPIOC_BASE+STM32_GPIO_OTYPER_OFFSET)
#define STM32_GPIOC_PUPDR        (STM32_GPIOC_BASE+STM32_GPIO_PUPDR_OFFSET)

/* AHB1 Peripheral Clock enable register */

#define RCC_AHB1ENR_GPIOCEN      (1 << 2)  /* Bit 2:  IO port C clock 
                                              enable */

/* GPIO port mode register */

#define GPIO_MODER_INPUT           (0) /* Input */
#define GPIO_MODER_OUTPUT          (1) /* General purpose output mode */
#define GPIO_MODER_ALT             (2) /* Alternate mode */
#define GPIO_MODER_ANALOG          (3) /* Analog mode */

#define GPIO_MODER13_SHIFT         (26)
#define GPIO_MODER13_MASK          (3 << GPIO_MODER13_SHIFT)

/* GPIO port output type register */

#define GPIO_OTYPER_PP             (0) /* 0=Output push-pull */
#define GPIO_OTYPER_OD             (1) /* 1=Output open-drain */

#define GPIO_OT13_SHIFT            (13)
#define GPIO_OT13_MASK             (1 << GPIO_OT13_SHIFT)

/* GPIO port pull-up/pull-down register */

#define GPIO_PUPDR_NONE            (0) /* No pull-up, pull-down */
#define GPIO_PUPDR_PULLUP          (1) /* Pull-up */
#define GPIO_PUPDR_PULLDOWN        (2) /* Pull-down */

#define GPIO_PUPDR13_SHIFT         (26)
#define GPIO_PUPDR13_MASK          (3 << GPIO_PUPDR13_SHIFT)

int main(int argc, char *argv[])
{
  uint32_t reg;

  /* Ponteiros para registradores */

  uint32_t *pRCC_AHB1ENR  = (uint32_t *)STM32_RCC_AHB1ENR;
  uint32_t *pGPIOC_MODER  = (uint32_t *)STM32_GPIOC_MODER;
  uint32_t *pGPIOC_OTYPER = (uint32_t *)STM32_GPIOC_OTYPER;
  uint32_t *pGPIOC_PUPDR  = (uint32_t *)STM32_GPIOC_PUPDR;
  uint32_t *pGPIOC_BSRR   = (uint32_t *)STM32_GPIOC_BSRR;

  /* Habilita clock GPIOC */

  reg  = *pRCC_AHB1ENR;
  reg |= RCC_AHB1ENR_GPIOCEN;
  *pRCC_AHB1ENR = reg;

  /* Configura PC13 como saida pull-up off e pull-down off */

  reg = *pGPIOC_MODER;
  reg &= ~(GPIO_MODER13_MASK);
  reg |= (GPIO_MODER_OUTPUT << GPIO_MODER13_SHIFT);
  *pGPIOC_MODER = reg;

  reg = *pGPIOC_OTYPER;
  reg &= ~(GPIO_OT13_MASK);
  reg |= (GPIO_OTYPER_PP << GPIO_OT13_SHIFT);
  *pGPIOC_OTYPER = reg;

  reg = *pGPIOC_PUPDR;
  reg &= ~(GPIO_PUPDR13_MASK);
  reg |= (GPIO_PUPDR_NONE << GPIO_PUPDR13_SHIFT);
  *pGPIOC_PUPDR = reg;

  while(1);

  /* Nao deveria chegar aqui */

  return EXIT_SUCCESS;
}
```
A partir de agora estamos aptos a alterar o estado do pino de saída. Para
alterar o estado de um pino de saída no STM32F411 temos duas alternativas. A
primeira é utilizar o registrador **GPIOx_ODR - GPIO port output data register**.
O principal inconveniente de utilizar este registrador é que é necessário ler o
estado dos outros pinos de saída, alterar apenas o valor do bit desejado e em
seguida escrever no registrador. Alternativamente, podemos utilizar o
registrador **GPIOx_BSRR - GPIO port bit set/reset register** que permite
alterar o estado do pino de saída de forma **atômica**.

Arquivo: **main.c**
```c
#include <stdlib.h>

/* AHB1 Base Addresses ******************************************************/

#define STM32_RCC_BASE       0x40023800     /* 0x40023800-0x40023bff: Reset 
                                               and Clock control RCC */

/* AHB2 Base Addresses ******************************************************/

#define STM32_GPIOC_BASE     0x48000800U    /* 0x48000800-0x48000bff: GPIO 
                                               Port C */

/* Register Offsets *********************************************************/

#define STM32_RCC_AHB1ENR_OFFSET  0x0030   /* AHB1 Peripheral Clock enable
                                               register */

#define STM32_GPIO_MODER_OFFSET   0x0000  /* GPIO port mode register */
#define STM32_GPIO_OTYPER_OFFSET  0x0004  /* GPIO port output type register */
#define STM32_GPIO_PUPDR_OFFSET   0x000c  /* GPIO port pull-up/pull-down
                                              register */
#define STM32_GPIO_BSRR_OFFSET    0x0018  /* GPIO port bit set/reset register */

/* Register Addresses *******************************************************/

#define STM32_RCC_AHB1ENR        (STM32_RCC_BASE+STM32_RCC_AHB1ENR_OFFSET)

#define STM32_GPIOC_MODER        (STM32_GPIOC_BASE+STM32_GPIO_MODER_OFFSET)
#define STM32_GPIOC_OTYPER       (STM32_GPIOC_BASE+STM32_GPIO_OTYPER_OFFSET)
#define STM32_GPIOC_PUPDR        (STM32_GPIOC_BASE+STM32_GPIO_PUPDR_OFFSET)
#define STM32_GPIOC_BSRR         (STM32_GPIOC_BASE + STM32_GPIO_BSRR_OFFSET)

.
.
.

/* GPIO port bit set/reset register */

#define GPIO_BSRR_SET(n)           (1 << (n))
#define GPIO_BSRR_RST(n)           (1 << (n + 16))

int main(int argc, char *argv[])
{
  uint32_t reg;

  /* Ponteiros para registradores */

  uint32_t *pRCC_AHB1ENR  = (uint32_t *)STM32_RCC_AHB1ENR;
  uint32_t *pGPIOC_MODER  = (uint32_t *)STM32_GPIOC_MODER;
  uint32_t *pGPIOC_OTYPER = (uint32_t *)STM32_GPIOC_OTYPER;
  uint32_t *pGPIOC_PUPDR  = (uint32_t *)STM32_GPIOC_PUPDR;
  uint32_t *pGPIOC_BSRR   = (uint32_t *)STM32_GPIOC_BSRR;

  /* Habilita clock GPIOC */

  .
  .
  .

  while(1)
    {
      *pGPIOC_BSRR = GPIO_BSRR_SET(13);
       for(uint32_t i = 0; i < LED_DELAY; i++);
      *pGPIOC_BSRR = GPIO_BSRR_RST(13);
       for(uint32_t i = 0; i < LED_DELAY; i++);
    }

  /* Nao deveria chegar aqui */

  return EXIT_SUCCESS;
}
```