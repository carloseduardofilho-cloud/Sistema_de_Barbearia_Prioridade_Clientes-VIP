# 💈 Sistema de Barbearia com Prioridade em Tempo Real (FreeRTOS + STM32)

Projeto da disciplina **Sistemas em Tempo Real**  

> **Integrantes**  
> Carlos Eduardo Alves Mamde Filho – 121110309  
> Ygor de Almeida Pereira – 121110166  

---

## 🔎 Descrição

Este projeto implementa uma variação do **Problema do Barbeiro Dorminhoco**, utilizando **FreeRTOS** em um microcontrolador **STM32F446RE**.  
O diferencial é a introdução de **clientes VIP**, que possuem prioridade absoluta sobre clientes normais, tornando o sistema uma aplicação de **tempo real hard**.

Além da lógica de escalonamento, o sistema foi **integrado ao hardware da PhotoBoard**:
- **LEDs**: representam o status das cadeiras e do barbeiro (ocupado, livre, dormindo).  
- **Botões**: geram interrupções simulando a chegada de clientes normais e VIPs.  

---

## 🎯 Motivação

A barbearia é usada como metáfora para problemas reais de **concorrência e tempo real**:
- **VIP ↔ tarefas críticas** (hard real-time).  
- **Clientes normais ↔ tarefas comuns** (soft real-time).  
- **Cadeiras ↔ buffers limitados**.  
- **Barbeiro ↔ recurso exclusivo**.  

Assim, o projeto demonstra como **escalonamento, semáforos e sincronização** podem ser aplicados em sistemas embarcados.

---

## ⚙️ Estrutura do Sistema

O sistema modela:

- **1 barbeiro**: atende apenas 1 cliente por vez.  
- **3 cadeiras de espera para clientes normais**.  
- **1 cadeira exclusiva para clientes VIP**.  
- **Geração de clientes**: feita via **interrupções externas** (botões).  

As tarefas são criadas no FreeRTOS:

```c
/* Task do Barbeiro */
osThreadId_t BarbeiroTaskHandle;
const osThreadAttr_t BarbeiroTask_attributes = {
  .name = "BarbeiroTask",
  .stack_size = 128 * 4,
  .priority = (osPriority_t) osPriorityAboveNormal,
};

/* Task de Geração de Clientes */
osThreadId_t CriaClienteTaskHandle;
const osThreadAttr_t CriaClienteTask_attributes = {
  .name = "CriaClienteTask",
  .stack_size = 128 * 4,
  .priority = (osPriority_t) osPriorityHigh,
};

```
---

## 🔗 Mecanismo de Sincronização

O sistema usa semáforos e filas para coordenar o fluxo:

- **sem_clientes_vip** → gerencia a cadeira VIP (máx. 1).  
- **sem_clientes_normais** → controla até 3 cadeiras normais.  
- **BarbeiroSem** → acorda o barbeiro quando um cliente chega.  
- **CorteSem e CorteVipSem** → sinalizam fim do atendimento.  

Exemplo simplificado:

```c
/* Semáforo para clientes VIP */
sem_clientes_vipHandle = osSemaphoreNew(1, 1, &sem_clientes_vip_attributes);

/* Semáforo para clientes normais */
sem_clientes_normaisHandle = osSemaphoreNew(3, 3, &sem_clientes_normais_attributes);

```

---

## 👨‍🔧 Funcionamento das Tarefas

### ✂️ Barbeiro
- Bloqueado em **BarbeiroSem** até cliente chegar.  
- Verifica primeiro se há **VIPs**; se não, atende **normais**.  
- Acende **LED correspondente** durante o corte.  

### 🙂 Cliente Normal
- Criado apenas se houver vaga nas **3 cadeiras**.  
- Caso contrário, "vai embora" (**LED piscando**).
  
### 👑 Cliente VIP
- Só acessa a **cadeira VIP**.  
- Sempre tem **prioridade no atendimento**.  
- Se a cadeira VIP estiver ocupada, **vai embora**.  

### 🔔 Geração de Clientes (Botões)
- Pressionar um botão gera **interrupção** que cria um cliente.  
- **LEDs indicam** a ocupação das cadeiras e fila.  

---

## 🛑 Desafios e Soluções

- **Prioridade Absoluta (VIP)**  
  VIPs sempre furam a fila de normais.  
  Resolvido com separação de semáforos e checagem explícita.  

- **Starvation de Normais**  
  Muitos VIPs poderiam atrasar indefinidamente normais.  
  Solução alternativa: alternância após certo número de VIPs.  

- **Deadlocks**  
  Situações de semáforo não liberado corrigidas com verificações extras.  

---

## 📊 Métricas Avaliadas
- Tempo médio de espera (**VIP vs. Normal**).  
- Clientes perdidos (**sem vaga → vão embora**).  
- Throughput do barbeiro (**cortes realizados / tempo**).  

---

## 💡 Integração com Hardware

**LEDs indicam:**
- Barbeiro dormindo/ocupado.  
- Cadeiras VIP/normal disponíveis ou ocupadas.  

**Botões simulam chegada de clientes:**
- Um botão cria **cliente normal**.  
- Outro botão cria **cliente VIP**.  

Exemplo no código (`main.c`):

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if(GPIO_Pin == BOTAO_NORMAL_Pin) {
        osThreadNew(ClientTask, (void*)NORMAL, &Cliente_attributes);
    } else if(GPIO_Pin == BOTAO_VIP_Pin) {
        osThreadNew(ClientTask, (void*)VIP, &Vip_attributes);
    }
}

```
---

## 🎥 Link do vídeo: 
