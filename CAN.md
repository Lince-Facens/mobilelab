---
title: Setting up the CAN Bus
permalink: /setting-up/can-bus/
---

# Setting up the Controller Area Network (CAN)

In the Mobilelab project we wanted to start using the CAN Bus to improve the reliability and speed of the communication.

## Hardware
![Hardware Schematic](img/schematic_can.svg)

## Software
### NVIDIA Jetson TX2
The Jetson TX2 supports two CAN ports, in our case we'll only use CAN0 located in the J26 header:
| J26 Pin | Description |
| :-----: | :---------: |
|       5 | CAN0 RX     |
|       7 | CAN0 TX     |

Luckily, the Jetson TX2 board supports the [SocketCAN](https://en.wikipedia.org/wiki/SocketCAN) API, which allows us to use tools and libraries such as [can-utils](https://github.com/linux-can/can-utils) and [python-can](https://github.com/hardbyte/python-can).

We'll start by enabling the module:
```bash
modprobe can
modprobe mttcan
```
To enable them on startup, add them into the `/etc/modules` file.

We can configure and enable the can0 pin, setting the bitrate to 1000000 bits per second (1MBps):
```bash
ip link set can0 type can bitrate 1000000
ip link set up can0
```

We can now analyse the bus using can-utils:
```bash
candump can0
```

### STM32F103C8T6

The STM32F103C8T6 board also comes with two CAN ports. In our case we are only going to use CAN1.
| STM32 Pin | Description |
| :-------: | :---------: |
|        B8 | CAN1 RX     |
|        B9 | CAN1 TX     |


#### NVIC configuration
```c
NVIC_InitTypeDef NVIC_InitStructure;

// Configure the CAN1 RX interruption
NVIC_InitStructure.NVIC_IRQChannel = USB_LP_CAN1_RX0_IRQn;
NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0x0;
NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0x0;
NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
NVIC_Init(&NVIC_InitStructure);
```

#### GPIO pin configuration
```c
GPIO_InitTypeDef GPIO_InitStructure;

// Enables AFIO and GPIOB
RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO | RCC_APB2Periph_GPIOB, ENABLE);

// Configure CAN RX pin (input)
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8; // B8
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
GPIO_Init(GPIOB, &GPIO_InitStructure);

// Configure CAN TX pin (output)
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9; // B9
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOB, &GPIO_InitStructure);

// Remaps the B8 and B9 pins to CAN
GPIO_PinRemapConfig(GPIO_Remap1_CAN1, ENABLE);
```

#### CAN configuration
```c
CAN_InitTypeDef CAN_InitStructure;

CAN_DeInit(CAN1);
CAN_StructInit(&CAN_InitStructure);

// Initializes CAN cell
CAN_InitStructure.CAN_TTCM = DISABLE; // Time triggered communication mode
CAN_InitStructure.CAN_ABOM = DISABLE; // Automatic bus-off management
CAN_InitStructure.CAN_AWUM = DISABLE; // Automatic wake-up mode
CAN_InitStructure.CAN_NART = DISABLE; // No-automatic retransmission mode
CAN_InitStructure.CAN_RFLM = DISABLE; // Receive FIFO Locked mode
CAN_InitStructure.CAN_TXFP = ENABLE; // Transmit FIFO priority
CAN_InitStructure.CAN_Mode = CAN_Mode_Normal; // Mode: Normal/Loopback/Silent/SilentLoopback
CAN_InitStructure.CAN_SJW = CAN_SJW_1tq; // Maximum of time quanta the hardware can shorten/lengthen a bit to resync
CAN_InitStructure.CAN_BS1 = CAN_BS1_3tq; // Number of time quanta in bit segment 1
CAN_InitStructure.CAN_BS2 = CAN_BS2_2tq; // Number of time quanta in bit segment 2

// The CAN bitrate in time quanta
// 6 = 1MBps
// 12 = 500KBps
// 24 = 250KBps
// 48 = 125KBps
// 60 = 100KBps
// 120 = 50KBps
// 300 = 20KBps
// 600 = 10KBps
CAN_InitStructure.CAN_Prescaler = 6;

CAN_Init(CAN1, &CAN_InitStructure);

// Enables FIFO #0
CAN_ITConfig(CAN1, CAN_IT_FMP0, ENABLE);
```

#### Configuring the filter
```c
CAN_FilterInitTypeDef CAN_FilterInitStructure;

CAN_FilterInitStructure.CAN_FilterNumber = 0; // The filter to initialize (ranges from 0 to 13)
CAN_FilterInitStructure.CAN_FilterMode = CAN_FilterMode_IdMask; // Whether it's an identifier mask or a list of identifiers
CAN_FilterInitStructure.CAN_FilterScale = CAN_FilterScale_32bit;
CAN_FilterInitStructure.CAN_FilterIdHigh = 0x0000; // MSB ID
CAN_FilterInitStructure.CAN_FilterIdLow = 0x0000; // LSB ID
CAN_FilterInitStructure.CAN_FilterMaskIdHigh = 0x0000; // MSB Mask ID
CAN_FilterInitStructure.CAN_FilterMaskIdLow = 0x0000; // LSB Mask ID
CAN_FilterInitStructure.CAN_FilterFIFOAssignment = 0; // Which FIFO will be assigned
CAN_FilterInitStructure.CAN_FilterActivation = ENABLE; // Enabled/Disabled
CAN_FilterInit(&CAN_FilterInitStructure);
```

#### Receiving messages
```c
CanRxMsg RxMessage;

// Interruption function
void USB_LP_CAN1_RX0_IRQHandler(void)
{
    CAN_Receive(CAN1, CAN_FIFO0, &RxMessage);
    // Treats the message received as the RxMessage object
    // ...
}
```

#### Sending messages
```c
CanTxMsg TxMessage;

TxMessage.StdId = 0x321; // CAN 2.0A ID
TxMessage.ExtId = 0x01; // CAN 2.0B ID
TxMessage.RTR = CAN_RTR_Data; // Whether it's a data frame or a remote frame
TxMessage.IDE = CAN_Id_Standard; // Whether it's a standard (CAN 2.0A) ID or an extended (CAN 2.0B) ID
TxMessage.DLC = 1; // The data length in bytes (up to 8 bytes)
TxMessage.Data[0] = 0x1F; // Sets the first byte value (ranges from 0x0 to 0xFF)

uint8_t mailbox = CAN_Transmit(CAN1, &TxMessage);

// Checks whether the message was sent to a mailbox
if (mailbox != CAN_TxStatus_NoMailBox) {
    uint8_t status;

    // Waits the message to finish transmitting
    while((status = CAN_TransmitStatus(CAN1, mailbox)) == CAN_TxStatus_Pending);
}
```


### Microchip CAN Bus Analyzer
On a Windows system, we can use the [official software](https://www.microchip.com/DevelopmentTools/ProductDetails/PartNO/APGDT002), but we found [SavvyCAN](https://www.savvycan.com/) to be a pretty good alternative for Linux that runs on top of the [SocketCAN](www.kernel.org/doc/html/v4.17/networking/can.html) drivers.

On Linux, we can also use the [can-utils](https://github.com/linux-can/can-utils) command line tool.

We can hook it up to the CAN_H and CAN_L signals to receive and transmit CAN messages.