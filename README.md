
# Tutorial: Sistema de Barbearia com Prioridade para Clientes VIP

**Universidade Federal de Campina Grande**  
Centro de Engenharia Elétrica e Informática  
Departamento de Engenharia Elétrica  
Disciplina: Sistemas em Tempo Real  
Professor: Kyller Costa Gorgonio

**Alunos:**  
- Carlos Eduardo Alves Mamede Filho — Matrícula: 121110309  
- Ygor de Almeida Pereira — Matrícula: 121110166

Campina Grande - PB, Setembro de 2025

## Sumário
1. [Especificações do Sistema](#especificações-do-sistema)  
   1.1 [Entendendo o Projeto](#entendendo-o-projeto)  
   1.2 [Componentes da Barbearia](#componentes-da-barbearia)  
   1.3 [Comportamento dos Clientes](#comportamento-dos-clientes)  
2. [Fluxo Geral](#fluxo-geral)  
3. [Guia de Implementação: STM32 + FreeRTOS](#guia-de-implementação-stm32--freertos)  
   3.1 [Estrutura do código](#estrutura-do-código)  
   3.1.1 [Inclusão de bibliotecas](#inclusão-de-bibliotecas)  
   3.1.2 [Estrutura de informações do cliente](#estrutura-de-informações-do-cliente)  
   3.1.3 [Definições de temporizador e variáveis privadas](#definições-de-temporizador-e-variáveis-privadas)  
   3.1.4 [Definição de tasks e semáforos do FreeRTOS](#definição-de-tasks-e-semáforos-do-freertos)  
   3.1.5 [Variáveis auxiliares e definição de cadeiras](#variáveis-auxiliares-e-definição-de-cadeiras)  
   3.1.6 [Protótipos de funções](#protótipos-de-funções)  
   3.1.7 [Inicialização do sistema e periféricos](#inicialização-do-sistema-e-periféricos)  
   3.1.8 [Criação e inicialização de semáforos](#criação-e-inicialização-de-semáforos)  
   3.1.9 [Criação de tasks do FreeRTOS](#criação-de-tasks-do-freertos)  
   3.1.10 [Inicialização do scheduler do FreeRTOS](#inicialização-do-scheduler-do-freertos)  
   3.1.11 [Configuração do clock do sistema](#configuração-do-clock-do-sistema)  
   3.1.12 [Configuração do Timer 13](#configuração-do-timer-13)  
   3.1.13 [Configuração da UART2](#configuração-da-uart2)  
   3.1.14 [Configuração das GPIOs](#configuração-das-gpios)  
   3.1.15 [Função CriaCliente](#função-criacliente)  
   3.1.16 [Função AcendeLed](#função-acendeled)  
   3.1.17 [Callback de interrupção dos botões](#callback-de-interrupção-dos-botões)  
   3.1.18 [Funções auxiliares de temporização e comunicação](#funções-auxiliares-de-temporização-e-comunicação)  
   3.1.19 [Task padrão StartDefaultTask](#task-padrão-startdefaulttask)  
   3.1.20 [Task de atendimento de cliente (StartCliente)](#task-de-atendimento-de-cliente-startcliente)  
   3.1.21 [Task de criação de clientes (StartCriaClienteTask)](#task-de-criação-de-clientes-startcriaclientetask)  
   3.1.22 [Task do barbeiro (StartBarbeiroTask)](#task-do-barbeiro-startbarbeirotask)  
   3.1.23 [Task de clientes VIP (StartVip)](#task-de-clientes-vip-startvip)  
   3.1.24 [Callbacks e tratamento de erros](#callbacks-e-tratamento-de-erros)  
4. [Componentes para Montagem do Protótipo](#componentes-para-montagem-do-protótipo)

---

## 1. Especificações do Sistema

### 1.1 Entendendo o Projeto
Este projeto é uma variação do clássico problema do *Sleeping Barber*, proposto por Edsger Dijkstra. O objetivo é implementar, em **FreeRTOS**, um sistema de gerenciamento de barbearia com suporte a concorrência, sincronização de processos e prioridades em tempo real.

Nesta versão são introduzidos **clientes VIP**, que possuem prioridade absoluta sobre clientes normais, preemptando a fila de espera. A implementação deve empregar mecanismos de sincronização (semaforos, mutexes, etc.), garantindo resposta imediata aos clientes, evitando deadlocks, minimizando starvation para clientes normais e assegurando restrições temporais (por exemplo, tempos máximos de espera menores para VIPs).

### 1.2 Componentes da Barbearia
- **1 barbeiro**: atende um cliente por vez e dorme quando não há clientes pendentes.  
- **1 cadeira de corte**: recurso compartilhado (exclusão mútua).  
- **3 cadeiras de espera**: para clientes normais. Se todas estiverem ocupadas, clientes normais vão embora.  
- **1 cadeira para clientes VIP**: exclusiva para VIPs. Se ocupada, VIPs vão embora.

### 1.3 Comportamento dos Clientes
- **Clientes normais**: chegam aleatoriamente, sentam nas cadeiras de espera se disponíveis. Esperam em fila (FIFO) para serem atendidos. Se a sala de espera estiver cheia, vão embora.  
- **Clientes VIP**: chegam aleatoriamente, sentam-se na cadeira VIP se disponível. Quando o barbeiro termina um atendimento, atende o VIP imediatamente (preemptando a fila de normais), mesmo que haja clientes normais esperando. Se a cadeira VIP estiver ocupada, vão embora.

---

## 2. Fluxo Geral
- O barbeiro dorme se não houver clientes (normais ou VIP).  

**Cliente normal:**
- Verifica cadeiras de espera livres.  
- Senta e acorda o barbeiro se ele estiver dormindo, ou espera na fila.  

**Cliente VIP:**
- Verifica cadeira VIP livre.  
- Senta e acorda o barbeiro se ele estiver dormindo.  
- É atendido na próxima oportunidade, furando a fila dos normais.  

**Após cada atendimento:**
- O barbeiro verifica primeiro se há VIP esperando; se não houver, atende o próximo normal da fila.

**Simulação de tempos e prioridades:**
- Chegada de clientes: aleatória.  
- Tempo de corte: fixo ou variável (ex.: 5–10 unidades de tempo).  
- Em tempo real: garantir resposta a VIPs em ≤ 1 unidade de tempo após barbeiro disponível.  
- **VIPs:** prioridade *hard real-time*.  
- **Normais:** prioridade *soft real-time* (evitar starvation).

**Métricas sugeridas:**
- Tempo médio de espera (normais e VIPs).  
- Taxa de clientes perdidos (que vão embora).  
- Throughput do barbeiro.

**Condições de erro a serem evitadas:**
- Race conditions.  
- Deadlocks.  
- Starvation (incluir mecanismos para atendimento eventual de normais).

---

## 3. Guia de Implementação: STM32 + FreeRTOS

### 3.1.1 Criação do Projeto na STM32CubeIDE

O STM32CubeIDE é uma plataforma avançada de desenvolvimento C/C++ com configuração de periféricos, geração de código, compilação de código e recursos de depuração para microcontroladores e microprocessadores STM32.

O primeiro passo na  STM32CubeIDE é **criar um novo projeto**, nesse caso no campo **board selector** escolhemos nossa placa que é a **NUCLEO-F446RE** a linguagem alvo é C, em seguida inicializamos os periféricos no **modo default**, esse modo traz alguns periféricos já configurados como a USART por exemplo. Por fim a IDE ira abrir uma janela de configuração do dispositivo, onde é possível visualizar os periféricos e os pinos do microprocessador.

<p align="center">
<img width="294" height="330" alt="image" src="https://github.com/user-attachments/assets/1b61d302-9c9a-42f9-b2a9-1e282a254cb4" />
</p>

<p align="center">
<img width="930" height="420" alt="image" src="https://github.com/user-attachments/assets/d90e7b78-c149-40b1-ba9a-efc074b63dbb" />
</p>

### 3.1.2 Configuração do FREERTOS
O primeiro passo é acessar, na barra lateral, o menu **System Core → SYS** e alterar a opção **Timebase Source**, substituindo o SysTick (que será utilizado pelo FreeRTOS) por um **timer (TIMx)**.

<p align="center">
<img width="1982" height="350" alt="image" src="https://github.com/user-attachments/assets/7efd0213-d80a-4619-9e90-23955654e2cb" />
</p>



Em seguida, no painel **Middleware and Software Packs**, habilite o **FreeRTOS** utilizando a interface **CMSIS_V2**. Ainda nessa janela, configure as tasks e os semáforos do sistema.
Na aba **Tasks and Queues**, já existe uma task padrão criada automaticamente pelo FreeRTOS. Para adicionar uma nova, clique em **Add**: será exibida uma janela onde é possível definir o nome da task, prioridade, tamanho da pilha, função associada e o tipo de alocação de memória.

<p align="center">
<img width="323" height="272" alt="image" src="https://github.com/user-attachments/assets/9c4bb0eb-1460-4368-a21f-9682c0c7045d" />
</p>

As seguintes tasks para esse projeto foram criadas conforme a imagem a seguir. Suas respectivas funções serão explicadas na sequência.

<p align="center">
<img width="636" height="150" alt="image" src="https://github.com/user-attachments/assets/19e7e2c3-ca83-485b-ac66-7567b0a906a9" />
</p>


O processo para configuração dos semáforos é similar, na aba **Timers and Semaphores** criamos os seguintes semafóros. Note que existem dois tipos de semafóros os binarios e os de contagem, os binários tem apenas um token de acesso, enquanto os de contagem podem ter mais de 1.

<p align="center">
<img width="638" height="260" alt="image" src="https://github.com/user-attachments/assets/cec346c2-8abf-4ac8-bdc7-218248c9e813" />
</p>

Por fim na janela **Advanced settings** habilite a configuração **USE_NEWLIB_REENTRANT**

<p align="center">
<img width="678" height="251" alt="image" src="https://github.com/user-attachments/assets/6190452c-6207-4ab7-8f09-33bc0f640e54" />
</p>


### 3.1.3  Configuração dos GPIO e interrupções 

O projeto utiliza LEDs e botões para permitir a interação física com o usuário e a visualização do funcionamento do código.
A configuração das portas **GPIO (General Purpose Input/Output)** é realizada por meio da visualização gráfica do microcontrolador e de seus pinos. Basta selecionar o pino desejado e atribuir a função correspondente, como **GPIO_Output**, **GPIO_Input** ou **GPIO_EXTI5**, entre outras.

Além disso, é possível definir **labels** (rótulos), que funcionam como “apelidos” para os pinos. Esses nomes facilitam a identificação e tornam o código mais legível.

Para este projeto, foram configurados os seguintes pinos GPIO:


<p align="center">
<img width="900" height="242" alt="image" src="https://github.com/user-attachments/assets/0e984e4c-cb3a-46c4-96c8-2e7063a3ffcc" />
</p>

<p align="center">
<img width="621" height="430" alt="image" src="https://github.com/user-attachments/assets/207e14cc-2830-4c9d-b515-4a5f7b8b84c5" />
</p>


Observe que a placa tem um botão próprio e um led PC13 PA5 respectivamente, ambos pré configurados por default, o nosso botão externo está no PC13 e recebe o modo **_EXIT (External Interrupt Mode)_** com uma configuração de **Pull-up**, assim a placa acrescenta um resistor interno de Pull-Up ao pino em que que ficara o botão. É **fundamental que se tenha um resistor de Pull-Up ou Pull-Down, interno ou externo para limitar a corrente e definir o estado de leitura do botão.** 
Para habilitar as interrupções, siga para **System Core->NVIC** e habilite os campos **EXTI line[i:j]** referente a numeração dos pinos GPIOs escolhidos, nesse caso os dois botões funcionarão como interrupções portanto habilitamos da seguinte forma:

<p align="center">
<img width="1130" height="596" alt="image" src="https://github.com/user-attachments/assets/8f835d7e-e485-4822-8aa8-e3942c0a2287" />
</p>



### 3.1.4 Configuração do Timer 

Uma das funcionalidades implementadas no código é a medição do tempo que o cliente leva desde a chegada até o término do atendimento. Para isso, utilizamos um **periférico temporizador (Timer)**.

Antes, porém, é importante compreender o funcionamento do **Timer** no STM32. De acordo com a configuração de clock padrão da placa, os temporizadores operam a **84 MHz**.

<p align="center">
<img width="1700" height="651" alt="image" src="https://github.com/user-attachments/assets/966cc9be-18c1-483a-86e1-4d8dcae35dcb" />
</p>

A frequência real de operação de um timer depende do valor configurado no registrador de **Prescaler (PSC)** e no **Counter Period (Auto-Reload Register – ARR).**
Neste projeto, utilizamos **PSC = ARR = 65535**, o que resulta em uma frequência de ticks de aproximadamente **1281 Hz**, correspondendo a um tempo máximo de contagem de cerca de **51 segundos**.

$$
f_{\text{timer}} = \dfrac{f_{\mathrm{clk\_timer}}}{(PSC+1)}
$$

$$
T_{\text{max}} = \dfrac{(ARR+1)}{f_{\mathrm{timer}}}
$$

Então na janela de timers selecionamos o **TIM13** e o ativamos, também configurando seus registradores da seguinte forma:

<p align="center">
<img width="884" height="538" alt="image" src="https://github.com/user-attachments/assets/0eddeae4-2679-49c5-98ba-0e3998df89d7" />
</p>

### 3.1.5 Configuração da USART 

Para transmitir as informações referentes ao tempo de atendimento de cada cliente, utilizamos a **comunicação serial via USART**. Neste projeto, a USART2 já vem configurada por default.
É importante, entretanto, verificar e ajustar corretamente os parâmetros básicos da comunicação, em especial o **Baud Rate**, para garantir compatibilidade e sincronização com o dispositivo ou programa que irá receber os dados. Aqui nesse tutorial vamos utilizar o **Hercules 3-2-8** para estabelecer a comunicação serial com o microcontrolador.

<p align="center">
<img width="824" height="550" alt="image" src="https://github.com/user-attachments/assets/533a7ebe-34ae-4e61-ba5c-18389aa6a605" />
</p>

### 3.1.6 Geração do código
Ao fim de todo este setup inicial, a interface pode gerar o código correspondente as configurações feitas ao **clicar no botão de geração de arquivo**.

<p align="center">
<img width="680" height="71" alt="image" src="https://github.com/user-attachments/assets/ff336ed9-4cd1-4663-a4d7-5f47c35c0fca" />
</p>

O seu projeto da IDE corresponde a uma pasta com diversas subpastas e arquivos, sua main.c está localizada em **Core -> Src -> main.c**, outras pastas carregam os includes e configurações nativas, o arquivo .ioc corresponde a interface gráfica que estavamos interagindo.

<p align="center">

<img width="184" height="306" alt="image" src="https://github.com/user-attachments/assets/55a907fc-8335-4025-8dcc-3c2403ffc8b4" />
</p>

### 3.2.1 Inclusão de bibliotecas

O projeto inicia com a inclusão de cabeçalhos essenciais. **main.h** contém definições gerais do
sistema, enquanto **cmsis_os.h** fornece acesso às funções do FreeRTOS via CMSIS-RTOS. Também
são usadas bibliotecas padrão do C: **stdlib.h** para alocação dinâmica e utilitários, e **stdio.h** para
entrada e saída, como printf() para depuração.


```c
#include "main.h"
#include "cmsis_os.h"
#include "stdlib.h"
#include "stdio.h"
```

### 3.1.2 Estrutura de informações do cliente

Define-se a **struct ClientInfo**, que armazena informações essenciais de cada cliente:

• **id**: identificador único do cliente.

• **duracao**: tempo necessário de atendimento.

```c
typedef struct {
  uint32_t id;
  uint32_t duracao;
} ClientInfo;
```

### 3.2.3 Definições de temporizador e variáveis privadas

Constantes do temporizador e variáveis para periféricos:

   • **TIMER_TICK_S**: período de um tick em segundos (780 µs).
   
   • **TIMER_MAX_CNT**: valor máximo do contador do timer (16 bits).
   
   • **htim13**: handle do Timer 13.
   
   • **huart2**: handle da UART2 para comunicação serial.
   


```c
#define TIMER_TICK_S (0.00078f) // 780 us
#define TIMER_MAX_CNT (65535)

TIM_HandleTypeDef htim13;
UART_HandleTypeDef huart2;
```

### 3.2.4 Definição de tasks e semáforos do FreeRTOS

Handles e atributos de tasks e semáforos permitem que o FreeRTOS gerencie threads e sincronização:

• Cada task possui um handle (**osThreadId_t**) e atributos (**osThreadAttr_t**) com nome, pilha
e prioridade.

• Semáforos usam handle (**osSemaphoreId_t**) e atributos (**osSemaphoreAttr_t**) com nome do
objeto.

```c
/* Definitions for defaultTask */
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
  .name = "defaultTask",
  .stack_size = 128 * 4,
  .priority = (osPriority_t) osPriorityNormal,
};

/* Definitions for Cliente */
osThreadId_t ClienteHandle;
const osThreadAttr_t Cliente_attributes = {
  .name = "Cliente",
  .stack_size = 512 * 4,
  .priority = (osPriority_t) osPriorityLow,
};

/* Definitions for CriaClienteTask */
osThreadId_t CriaClienteTaskHandle;
const osThreadAttr_t CriaClienteTask_attributes = {
  .name = "CriaClienteTask",
  .stack_size = 128 * 4,
  .priority = (osPriority_t) osPriorityHigh,
};

/* Definitions for BarbeiroTask */
osThreadId_t BarbeiroTaskHandle;
const osThreadAttr_t BarbeiroTask_attributes = {
  .name = "BarbeiroTask",
  .stack_size = 128 * 4,
  .priority = (osPriority_t) osPriorityAboveNormal,
};

/* Definitions for Vip */
osThreadId_t VipHandle;
const osThreadAttr_t Vip_attributes = {
  .name = "Vip",
  .stack_size = 512 * 4,
  .priority = (osPriority_t) osPriorityLow,
};

/* Definitions for semaphores */
osSemaphoreId_t InterruptSemHandle;
const osSemaphoreAttr_t InterruptSem_attributes = {.name = "InterruptSem"};
osSemaphoreId_t BarbeiroSemHandle;
const osSemaphoreAttr_t BarbeiroSem_attributes = {.name = "BarbeiroSem"};
osSemaphoreId_t CorteSemHandle;
const osSemaphoreAttr_t CorteSem_attributes = {.name = "CorteSem"};
osSemaphoreId_t sem_clientes_vipHandle;
const osSemaphoreAttr_t sem_clientes_vip_attributes = {.name = "sem_clientes_vip"};
osSemaphoreId_t CorteVipSemHandle;
const osSemaphoreAttr_t CorteVipSem_attributes = {.name = "CorteVipSem"};
osSemaphoreId_t sem_clientes_normaisHandle;
const osSemaphoreAttr_t sem_clientes_normais_attributes = {.name = "sem_clientes_normais"};
```

### 3.2.5 Variáveis auxiliares e definição de cadeiras

• **tipo**: indica se o cliente é normal ou VIP.

• **buffer**: armazenamento temporário para mensagens ou comunicação via UART.

• **CADEIRAS_NORMAIS_LIVRES**: define o número de cadeiras de espera disponíveis para clientes
normais.

```c
uint8_t tipo;
char buffer[64];
#define CADEIRAS_NORMAIS_LIVRES 3
```

### 3.2.6 Protótipos de funções


Protótipos das funções de inicialização de hardware e das tasks do FreeRTOS, além de funções
auxiliares definidas pelo usuário:

• **Inicialização de hardware**: SystemClock_Config, MX_GPIO_Init, MX_USART2_UART_Init, MX_TIM13_Ini

• **Tasks do FreeRTOS**: StartDefaultTask, StartCliente, StartCriaClienteTask, StartBarbeiroTask,StartVip.

• **Funções auxiliares**: Task_action, CriaCliente, AcendeLed.

```c
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART2_UART_Init(void);
static void MX_TIM13_Init(void);
void StartDefaultTask(void *argument);
void StartCliente(void *argument);
void StartCriaClienteTask(void *argument);
void StartBarbeiroTask(void *argument);
void StartVip(void *argument);

void Task_action(char message);
void CriaCliente(uint32_t id_recebido, uint32_t duracao_recebido);
void AcendeLed(void);
```

### 3.2.7 Inicialização do sistema e periféricos

Sequência de inicialização do microcontrolador e periféricos antes do FreeRTOS:

1. **HAL_Init()**: inicializa a HAL e o SysTick.

2. **SystemClock_Config()**: configura relógio do sistema.

3. **Inicializa periféricos configurados**:

   • MX_GPIO_Init(): configura pinos de entrada/saída.

   • MX_USART2_UART_Init(): inicializa UART para comunicação serial.

   • MX_TIM13_Init(): inicializa Timer 13.

4. **HAL_TIM_Base_Start()**: inicia o timer base para eventos temporizados.

5. **osKernelInitialize()**: inicializa o kernel do FreeRTOS antes de criar tasks.

```c
HAL_Init();
SystemClock_Config();
MX_GPIO_Init();
MX_USART2_UART_Init();
MX_TIM13_Init();
HAL_TIM_Base_Start(&htim13);
osKernelInitialize();
```

### 3.2.8 Criação e inicialização de semáforos

Neste ponto do código, os semáforos do FreeRTOS são criados utilizando **osSemaphoreNew()**,
definindo quantos tokens estão disponíveis e quais atributos cada semáforo possui. Os semáforos são
essenciais para **controlar o acesso a recursos compartilhados** (como cadeiras e o barbeiro) e para
**sincronizar tarefas**, garantindo que VIPs e clientes normais sejam atendidos corretamente.

1. InterruptSemHandle: semáforo binário criado com 1 token máximo e 0 disponível inicialmente.
Usado para sinalizar eventos ou interrupções.

2. BarbeiroSemHandle: semáforo binário para controlar quando o barbeiro está disponível. Inicialmente 0, sinaliza que o barbeiro pode atender.

3. CorteSemHandle: semáforo binário para coordenar o início do corte de cabelo entre barbeiro e
cliente.

4. sem_clientes_vipHandle: semáforo binário para controlar a presença de clientes VIP. Inicialmente 1 token disponível, permitindo que o VIP seja atendido imediatamente.

5. CorteVipSemHandle: semáforo binário usado para controlar o corte de VIPs, evitando conflitos
com a fila de normais.

6. sem_clientes_normaisHandle: semáforo contado (3 tokens) para controlar o acesso das cadeiras de espera normais. Cada token representa uma cadeira livre.

```c
/* creation of InterruptSem */
InterruptSemHandle = osSemaphoreNew(1, 0, &InterruptSem_attributes);

/* creation of BarbeiroSem */
BarbeiroSemHandle = osSemaphoreNew(1, 0, &BarbeiroSem_attributes);

/* creation of CorteSem */
CorteSemHandle = osSemaphoreNew(1, 0, &CorteSem_attributes);

/* creation of sem_clientes_vip */
sem_clientes_vipHandle = osSemaphoreNew(1, 1, &sem_clientes_vip_attributes);

/* creation of CorteVipSem */
CorteVipSemHandle = osSemaphoreNew(1, 0, &CorteVipSem_attributes);

/* creation of sem_clientes_normais */
sem_clientes_normaisHandle = osSemaphoreNew(3, 3, &sem_clientes_normais_attributes);
```

### 3.2.9 Criação de tasks do FreeRTOS

Neste trecho, as tasks principais do sistema são criadas utilizando a função osThreadNew().
Cada task representa uma thread independente no FreeRTOS, com sua própria pilha, prioridade e
função de execução. A criação correta das tasks é crucial para garantir a **sincronização, concorrência
e prioridade de atendimento**, especialmente para clientes VIP e normais.

1. CriaClienteTaskHandle:

• Criada a partir da função StartCriaClienteTask.

• Responsável por gerar clientes aleatórios (normais e VIP) durante a simulação.

• Prioridade alta, garantindo que a criação de clientes seja processada rapidamente.

2. BarbeiroTaskHandle:

• Criada a partir da função StartBarbeiroTask.

• Representa a lógica do barbeiro, que atende clientes conforme semáforos e prioridades.

• Prioridade acima do normal, permitindo que o barbeiro responda rapidamente à chegada
de VIPs e clientes normais.

```c
/* creation of CriaClienteTask */
CriaClienteTaskHandle = osThreadNew(StartCriaClienteTask, NULL, &CriaClienteTask_attributes);

/* creation of BarbeiroTask */
BarbeiroTaskHandle = osThreadNew(StartBarbeiroTask, NULL, &BarbeiroTask_attributes);
```

### 3.2.10 Inicialização do scheduler do FreeRTOS
Após criar todas as tasks e semáforos, o código chama:

```c
osKernelStart();
```


### 3.2.11 Função CriaCliente

A função **CriaCliente()** é chamada quando um botão é pressionado, por meio de uma **interrupção externa**. Seu objetivo é criar dinamicamente um cliente e alocá-lo na fila correta (normal
ou VIP), respeitando as regras de disponibilidade de cadeiras.

1. **Alocação de memória:**

• Cria um ponteiro ClientInfo *novoCliente e aloca memória usando malloc.

• Inicializa os campos id e duracao com os valores recebidos como parâmetros.

2. **Verificação do tipo de cliente:**

• Se tipo == 0, é cliente normal.

• Caso contrário, é cliente VIP.

3.** Controle de acesso com semáforos:**

• Para clientes normais: tenta adquirir o semáforo sem_clientes_normaisHandle com timeout 0.

– Se conseguir, cria uma task StartCliente para o cliente.

– Se não houver vaga, libera a memória com free(novoCliente).

• Para clientes VIP: verifica sem_clientes_vipHandle com timeout 0.

– Se disponível, cria a task StartVip.

– Caso contrário, libera a memória.


```c
void CriaCliente(uint32_t id_recebido, uint32_t duracao_recebido) {
  // aloca espaço para os dados do cliente
  ClientInfo *novoCliente = malloc(sizeof(ClientInfo));

  novoCliente->id = id_recebido;
  novoCliente->duracao = duracao_recebido;

  // Se cliente normal
  if(tipo == 0) {
    if(osSemaphoreAcquire(sem_clientes_normaisHandle, 0) == osOK) {
      osThreadNew(StartCliente, novoCliente, &Cliente_attributes);
    } else {
      free(novoCliente);
    }
  }
  // Se cliente VIP
  else {
    if(osSemaphoreAcquire(sem_clientes_vipHandle, 0) == osOK) {
      osThreadNew(StartVip, novoCliente, &Vip_attributes);
    } else {
      free(novoCliente);
    }
  }
}
```

### 3.1.12 Função AcendeLed

Esta função atualiza LEDs para indicar visualmente a ocupação das cadeiras na barbearia:

• Lê o número de cadeiras disponíveis para clientes normais e VIP usando **osSemaphoreGetCount()**.

• Acende ou apaga LEDs conectados aos pinos GPIO de acordo com a ocupação:

– Mais clientes sentados → mais LEDs acesos.
– Todas as cadeiras livres → LEDs apagados.

• Um LED separado indica se a cadeira VIP está ocupada (PC7 aceso quando VIP presente).

• Permite **monitoramento visual rápido** do status do sistema sem depuração via terminal.


```c
void AcendeLed(void) {
  uint8_t count = osSemaphoreGetCount(sem_clientes_normaisHandle);
  uint8_t count_vip = osSemaphoreGetCount(sem_clientes_vipHandle);

  if (count == 3) {
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, 0);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_9, 0);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, 0);
  }
  else if (count == 2) {
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, 1);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, 0);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_9, 0);
  }
  else if (count == 1) {
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, 1);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, 1);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_9, 0);
  }
  else if (count == 0) {
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_8, 1);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_9, 1);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, 1);
  }

  if (count_vip == 0) {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_7, 1);
  } else {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_7, 0);
  }
}
```

### 3.1.13 Callback de interrupção dos botões

Esta função é chamada automaticamente pelo HAL quando ocorre uma interrupção nos pinos
configurados como **EXTI (botões)**. Como a execução de uma interrupção deve ser rápida, não chamamos diretamente a task; usamos semáforos e uma variável tipo para sinalizar o tipo de cliente a ser
criado.

• **GPIO_Pin == B1_Pin**: botão para cliente normal. - Libera o semáforo InterruptSemHandle,
sinalizando que há um cliente para processar. - Define tipo = 0, indicando cliente normal.

• **GPIO_Pin == B5_Pin**: botão para cliente VIP. - Também libera InterruptSemHandle. - Define
tipo = 1, indicando cliente VIP.


```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
  if (GPIO_Pin == B1_Pin) {
    osSemaphoreRelease(InterruptSemHandle);
    tipo = 0;
  }
  if (GPIO_Pin == B5_Pin) {
    osSemaphoreRelease(InterruptSemHandle);
    tipo = 1;
  }
}
```

### 3.1.14 Funções auxiliares de temporização e comunicação
O projeto inclui funções auxiliares para calcular tempo decorrido usando o timer e enviar
mensagens via UART.

• **Timer_GetElapsedTime**: calcula o tempo decorrido desde um contador inicial do timer.

– Lê o valor atual do timer com __HAL_TIM_GET_COUNTER.

– Calcula a diferença considerando overflow do timer.

– Converte ticks para segundos multiplicando por TIMER_TICK_S.

• **UART_SendString**: envia uma string via UART. - Converte a string para uint8_t* e transmite
usando HAL_UART_Transmit. - Usa HAL_MAX_DELAY para aguardar até a transmissão completar.

```c
float Timer_GetElapsedTime(TIM_HandleTypeDef *htim, uint32_t start) {
  uint32_t end = __HAL_TIM_GET_COUNTER(htim);
  uint32_t elapsed_ticks;

  if (end >= start) {
    elapsed_ticks = end - start;
  } else {
    elapsed_ticks = (TIMER_MAX_CNT - start) + end + 1;
  }

  return elapsed_ticks * TIMER_TICK_S; // retorna em segundos
}

void UART_SendString(UART_HandleTypeDef *huart, const char *msg) {
  HAL_UART_Transmit(huart, (uint8_t *)msg, strlen(msg), HAL_MAX_DELAY);
}
```

### 3.1.15 Task padrão StartDefaultTask

O STM32CubeMX gera automaticamente uma task padrão chamada defaultTask. Ela serve
como um loop contínuo que pode ser usado para inicializações ou monitoramento, mas no nosso
projeto não realiza nenhuma ação específica.

• O loop infinito (for(;;)) mantém a task em execução contínua.

• osDelay(1) faz a task aguardar 1 tick do RTOS, evitando ocupação total da CPU e permitindo
que outras tasks sejam executadas.

• No contexto do projeto da barbearia, essa task não interfere nas tasks críticas (Cliente, Barbeiro,
CriaClienteTask e Vip).

```c
void StartDefaultTask(void *argument) {
  /* Infinite loop */
  for (;;) {
    osDelay(1);
  }
}
```

### 3.1.16 Task de atendimento de cliente (StartCliente)

Esta task representa o atendimento de um cliente normal pelo barbeiro. Cada cliente cria
uma instância desta task com seus próprios dados.

• O argumento recebido é um ponteiro void*, convertido para ClientInfo* para acessar informações do cliente (id e duracao).

• Captura o tempo inicial do timer com __HAL_TIM_GET_COUNTER para medir duração do corte.

• Libera o semáforo BarbeiroSemHandle para "acordar"o barbeiro.

• Aguarda a finalização do corte com osSemaphoreAcquire(CorteSemHandle, osWaitForever),

bloqueando a task até que o barbeiro termine.

• Calcula o tempo decorrido usando Timer_GetElapsedTime e envia mensagem via UART mostrando a duração do atendimento.

• Libera a memória alocada dinamicamente para este cliente com free(info).

• Finaliza a task chamando osThreadExit().

```c
void StartCliente(void *argument) {
  ClientInfo *info = (ClientInfo *) argument; // Carrega os dados do cliente
  uint32_t numero = info->id;

  uint32_t start = __HAL_TIM_GET_COUNTER(&htim13);
  osSemaphoreRelease(BarbeiroSemHandle); // Acorda o barbeiro
  osSemaphoreAcquire(CorteSemHandle, osWaitForever); // Espera pelo corte acabar

  float elapsed_s = Timer_GetElapsedTime(&htim13, start);
  snprintf(buffer, sizeof(buffer), "Tempo decorrido: %.2f s\r\n", elapsed_s);
  UART_SendString(&huart2, buffer);

  free(info); // Libera memória
  osThreadExit(); // Encerra a task
}
```

### 3.1.17 Task de criação de clientes (StartCriaClienteTask)

Esta task é responsável por criar novos clientes (normais ou VIP) quando ocorre uma interrupção de botão, simulando a chegada de clientes na barbearia.

• Inicializa um id para identificar cada cliente e define uma duração fixa de atendimento (duracao
= 3000 ms).

• Executa um loop infinito (for(;;)), pois a task deve permanecer ativa durante toda a execução
do sistema.

• Aguarda o semáforo InterruptSemHandle ser liberado (osSemaphoreAcquire(..., osWaitForever)),
o que acontece quando um botão é pressionado (interrupção).

• Chama a função CriaCliente(id, duracao) para criar a task do cliente correspondente (normal ou VIP).

• Atualiza os LEDs chamando AcendeLed() para indicar visualmente o status das cadeiras.

• Incrementa o id para o próximo cliente, garantindo identificação única.

```c
void StartCriaClienteTask(void *argument) {
  uint32_t id = 0;
  uint32_t duracao = 3000;

  for (;;) {
    osSemaphoreAcquire(InterruptSemHandle, osWaitForever);
    CriaCliente(id, duracao);

    AcendeLed();
    id++;
  }
}
```

### 3.1.18 Task do barbeiro (StartBarbeiroTask)

Esta task representa o comportamento do barbeiro, incluindo o atendimento de clientes normais e VIPs com prioridade, controlando LEDs e semáforos.

• Executa um loop infinito, mantendo o barbeiro sempre ativo enquanto houver clientes.

• Espera ser "acordado"por um cliente, bloqueando a execução em **osSemaphoreAcquire(BarbeiroSemHanosWaitForever)** até que um cliente chegue.

• Liga um **LED (GPIOA, PIN_10)** para indicar que o barbeiro está ocupado.

• Lê o número de clientes esperando: count_vip para VIPs e count para normais, usando
**osSemaphoreGetCount**.

• Se houver clientes VIP (count_vip == 0 significa cadeira VIP ocupada):

– Libera o semáforo do cliente VIP (sem_clientes_vipHandle) para iniciar o atendimento.

– Atualiza LEDs chamando AcendeLed().

– Aguarda 5 segundos (osDelay(5000)) simulando o corte.

– Libera CorteVipSemHandle para sinalizar que o atendimento VIP terminou.

• Caso contrário, atende um cliente normal:

– Libera o semáforo do cliente normal (sem_clientes_normaisHandle).

– Atualiza LEDs.

– Aguarda 5 segundos simulando o corte.

– Libera CorteSemHandle indicando fim do corte normal.

• Após o atendimento, verifica se ainda há clientes na fila:

– Se houver clientes esperando, libera novamente **BarbeiroSemHandle** para continuar o atendimento.

– Caso contrário, desliga o LED do barbeiro (GPIOA, PIN_10), indicando que ele está dormindo.

```c
void StartBarbeiroTask(void *argument) {
  uint16_t count, count_vip;

  for (;;) {
    osSemaphoreAcquire(BarbeiroSemHandle, osWaitForever); // Barbeiro espera acordar
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_10, 1);

    count_vip = osSemaphoreGetCount(sem_clientes_vipHandle);
    count = osSemaphoreGetCount(sem_clientes_normaisHandle);

    if (count_vip == 0) {
      osSemaphoreRelease(sem_clientes_vipHandle); // Libera um VIP da fila
      AcendeLed();
      osDelay(5000); // Realizando o corte
      osSemaphoreRelease(CorteVipSemHandle); // Corte VIP finalizado
    } else {
      osSemaphoreRelease(sem_clientes_normaisHandle); // Libera um cliente normal
      AcendeLed();
      osDelay(5000); // Realizando o corte
      osSemaphoreRelease(CorteSemHandle); // Corte normal finalizado
    }

    count_vip = osSemaphoreGetCount(sem_clientes_vipHandle);
    count = osSemaphoreGetCount(sem_clientes_normaisHandle);

    if (count != 3 || count_vip != 1) {
      osSemaphoreRelease(BarbeiroSemHandle); // Continua atendendo se houver clientes
    } else {
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_10, 0); // Barbeiro dorme
    }
  }
}
```

### 3.1.23 Task de clientes VIP (StartVip)

Esta task representa o atendimento de um cliente VIP, garantindo prioridade máxima sobre
os clientes normais e controle de tempo do atendimento.

• Recebe um ponteiro genérico **(void *argument)** e faz **cast** para **ClientInfo*** para acessar os
dados do cliente, como **id** e **duracao**.

• Captura o tempo inicial do timer (**__HAL_TIM_GET_COUNTER**) para medir o tempo de espera e
atendimento.
• Libera o semáforo **BarbeiroSemHandle**, acordando o barbeiro caso ele esteja dormindo.

• Aguarda o término do corte VIP com **osSemaphoreAcquire(CorteVipSemHandle, osWaitForever)**,
garantindo que a task só continue quando o barbeiro finalizar o atendimento.

• Calcula o tempo decorrido usando **Timer_GetElapsedTime** e envia uma mensagem via UART
com o tempo total do atendimento VIP.

• Libera a memória alocada para o cliente com **free(info)** para evitar vazamentos de memória.

• Finaliza a task com **osThreadExit()**, encerrando a execução do cliente VIP.

```c
void StartVip(void *argument) {
  ClientInfo *info = (ClientInfo *) argument; // Carrega os dados do cliente

  uint32_t start = __HAL_TIM_GET_COUNTER(&htim13);
  osSemaphoreRelease(BarbeiroSemHandle); // Acorda o barbeiro
  osSemaphoreAcquire(CorteVipSemHandle, osWaitForever); // Espera pelo corte VIP

  // Corte finalizado
  float elapsed_s = Timer_GetElapsedTime(&htim13, start);
  snprintf(buffer, sizeof(buffer), "Tempo decorrido VIP: %.2f s\r\n", elapsed_s);
  UART_SendString(&huart2, buffer);

  free(info); // Libera a memória do cliente
  osThreadExit(); // Encerra a task
}
```

---

## 4. Componentes para Montagem do Protótipo

| Componente       | Quantidade |
|------------------|------------|
| Protoboard       | 1          |
| LED              | 5          |
| Resistor 330Ω    | 5          |
| Botão            | 1          |
| Fios de conexão  | Diversos   |

> Figura 1: Protótipo do sistema de barbearia montado em protoboard com STM32. (ver PDF original)

---

## Observações finais
- Projeto desenvolvido como prática da disciplina **Sistemas em Tempo Real**.  
- Baseado no problema clássico do *Sleeping Barber* com adaptação para **prioridade VIP**.  
- Código e configurações (clocks, timers, GPIOs, UART, semáforos e tasks) exemplificados conforme o relatório original.

---

Se quiser, eu:
- salvo este `README.md` pronto para você fazer upload no repositório (já salvo neste workspace),  
- ou faço ajustes (reduzir/incluir trechos de código, adicionar imagens, badges do GitHub, exemplos de uso, etc).

