# STM32 Blue Pill USART1 RX with DMA

This project shows how to receive UART data on an **STM32F103 Blue Pill** using **USART1 RX with DMA** and `HAL_UARTEx_ReceiveToIdle_DMA()`.

The STM32 receives data through UART, detects the end of the message using the UART **IDLE line**, copies the received bytes, then sends them back using UART transmit.

---

## Hardware setup

| STM32 Blue Pill | USB-to-UART adapter |
|---|---|
| PA9 / USART1_TX | RX |
| PA10 / USART1_RX | TX |
| GND | GND |

Important:

```text
USART1_TX = PA9
USART1_RX = PA10
GND must be common
UART voltage must be 3.3 V logic
Do not use RS232 ±12 V directly
```

---

## CubeMX setup

### 1. Enable USART1

Go to:

```text
Connectivity → USART1
```

Set:

```text
Mode: Asynchronous
Baud rate: 115200
Word length: 8 bits
Parity: None
Stop bits: 1
Hardware flow control: None
```

CubeMX should assign:

```text
PA9  → USART1_TX
PA10 → USART1_RX
```

---

### 2. Enable DMA for USART1 RX

Go to:

```text
USART1 → DMA Settings → Add
```

Select:

```text
USART1_RX
```

For STM32F103 Blue Pill, USART1 RX normally uses:

```text
DMA1 Channel 5
```

Configure DMA like this:

```text
Direction: Peripheral to Memory
Mode: Normal
Peripheral Increment: Disabled
Memory Increment: Enabled
Peripheral Data Width: Byte
Memory Data Width: Byte
Priority: High
```

---

### 3. Enable interrupts

Go to:

```text
System Core → NVIC
```

Enable:

```text
DMA1 Channel5 global interrupt
USART1 global interrupt
```

The USART1 interrupt is needed because `HAL_UARTEx_ReceiveToIdle_DMA()` uses the UART IDLE event to detect the end of a variable-length message.

---

## Code

```c
#include "main.h"
#include <string.h>
#include <stdio.h>

#define UART_RX_DMA_SIZE 256

uint8_t uart_rx_dma[UART_RX_DMA_SIZE];
uint8_t uart_rx_copy[UART_RX_DMA_SIZE];

volatile uint8_t uart_rx_ready = 0;
volatile uint16_t uart_rx_size = 0;

UART_HandleTypeDef huart1;
DMA_HandleTypeDef hdma_usart1_rx;

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_DMA_Init();
    MX_USART1_UART_Init();

    HAL_UARTEx_ReceiveToIdle_DMA(&huart1, uart_rx_dma, UART_RX_DMA_SIZE);

    while (1)
    {
        if (uart_rx_ready)
        {
            uart_rx_ready = 0;

            HAL_UART_Transmit(&huart1, uart_rx_copy, uart_rx_size, HAL_MAX_DELAY);
        }
    }
}
```

---

## RX callback

```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size)
{
    if (huart->Instance == USART1)
    {
        memcpy(uart_rx_copy, uart_rx_dma, Size);

        uart_rx_size = Size;
        uart_rx_ready = 1;

        HAL_UARTEx_ReceiveToIdle_DMA(&huart1, uart_rx_dma, UART_RX_DMA_SIZE);
    }
}
```

---

## How the code works

### 1. Start UART reception with DMA

```c
HAL_UARTEx_ReceiveToIdle_DMA(&huart1, uart_rx_dma, UART_RX_DMA_SIZE);
```

This tells the STM32 to receive UART data using DMA.

Received bytes are stored automatically inside:

```c
uart_rx_dma
```

The CPU does not need to handle every received byte manually.

---

### 2. IDLE line detects the end of the message

UART messages are often variable-length.

`HAL_UARTEx_ReceiveToIdle_DMA()` calls this callback when the UART line becomes idle:

```c
HAL_UARTEx_RxEventCallback()
```

The argument `Size` tells how many bytes were received.

---

### 3. Copy received data

Inside the callback:

```c
memcpy(uart_rx_copy, uart_rx_dma, Size);
```

The received data is copied from the DMA buffer to another buffer.

This is useful because the DMA buffer will be reused for the next reception.

---

### 4. Tell the main loop data is ready

```c
uart_rx_size = Size;
uart_rx_ready = 1;
```

The callback only prepares the data.

The real processing is done in the `while (1)` loop.

---

### 5. Echo the received data

In the main loop:

```c
HAL_UART_Transmit(&huart1, uart_rx_copy, uart_rx_size, HAL_MAX_DELAY);
```

The STM32 sends back the received bytes.

So if you send:

```text
hello
```

The STM32 replies:

```text
hello
```

---

### 6. Restart DMA reception

At the end of the callback:

```c
HAL_UARTEx_ReceiveToIdle_DMA(&huart1, uart_rx_dma, UART_RX_DMA_SIZE);
```

This restarts UART DMA reception for the next message.
