## 1.Enable PA

### HAL\Target\Config\hal_board_cfg.h

```
#define xHAL_PA_LNA
```
change to
```
#define HAL_PA_LNA
```

*After enabling HAL_PA_LNA, compiling may result in error about segment XDATA is too long. Go to Project Options (right click on project name then options) -> General Options -> Stack/Heap -> Stack XDATA size, should be 0x4D0 or something, decrease it by the required space (should not be too large, in my case just 1 byte).

### MAC\Low Level\System\mac_radio_defs.c -> macRadioTurnOnPower()

Code between comments "...during sleep." and "For any RX...", only keep
```
       /* P1_1 -> PAEN */
      RFC_OBS_CTRL0 = RFC_OBS_CTRL_PA_PD_INV;
      OBSSEL1       = OBSSEL_OBS_CTRL0;
      
      /* P1_0 -> EN (LNA control) */
      RFC_OBS_CTRL1 = RFC_OBS_CTRL_LNAMIX_PD_INV;
      OBSSEL0       = OBSSEL_OBS_CTRL1;
```

### MAC\High Level\mac_pib.c

Field before "/* phyTransmitPower */", change to
```
#if defined (HAL_PA_LNA)
  27,                                          /* phyTransmitPower */
#else
  0,
#endif
```

## 2. Switch UART0 from DMA mode to ISR mode (for RS485 direction control)

### HAL\Target\Config\hal_board_cfg.h

Change both to DMA=0 ISR=1

```
#if defined HAL_SB_BOOT_CODE
#define HAL_UART_DMA  0
#define HAL_UART_ISR  1
#else
#define HAL_UART_DMA  0
#define HAL_UART_ISR  1
#endif
#define HAL_UART_USB  0
```

### HAL\Target\CC2530ZNP\Drivers\hal_dma.c

Comment the two lines about `HalUARTIsrDMA`

```
HAL_ISR_FUNCTION( halDmaIsr, DMA_VECTOR )
{
  HAL_ENTER_ISR();

  DMAIF = 0;

  if (ZNP_CFG1_UART == znpCfg1)
  {
    if (HAL_DMA_CHECK_IRQ(HAL_DMA_CH_TX))
    {
      //extern void HalUARTIsrDMA(void);        // <--Comment!!!
      //HalUARTIsrDMA();        // <--Comment!!!
    }
  }
#if (defined HAL_SPI) && (HAL_SPI == TRUE)
...
```

## 3. Switch UART0 from P0.2/P0.3 to P1.4/P1.5

### HAL\Target\CC2530ZNP\Drivers\hal_uart.c -> _hal_uart_isr.c -> HalUARTInitISR()

Comment all but the line setting bit 0 of `PERCFG` to 1 in the block below:

```
//#if (HAL_UART_ISR == 1)
  //PERCFG &= ~HAL_UART_PERCFG_BIT;    // Set UART0 I/O location to P0.
//#else
  PERCFG |= HAL_UART_PERCFG_BIT;     // Set UART1 I/O location to P1.      <--Only keep this line!!!
//#endif
```

## 4. Toggle RUN_LED (P1.3) according to UART state, also disable NWK_LED since we are not using it

### HAL\Target\Config\hal_board_cfg.h

Change LED1 definition to P1.2 active low, modify code as below.

```
/* ------------------------------------------------------------------------------------------------
 *                                       LED Configuration
 * ------------------------------------------------------------------------------------------------
 */

...

/* 1 - Green */
#define LED1_BV           BV(2)
#define LED1_SBIT         P1_2
#define LED1_DDR          P1DIR
#define LED1_POLARITY     ACTIVE_LOW
```

Add some code to `HAL_BOARD_INIT()` in order to initialize RUN_LED P1.3, and turn off NWK_LED P1.2. **Pay attention to the lines with comment "Add: ..."**

```
#define HAL_BOARD_INIT() st                                      \
(                                                                \
...
  /* set direction for GPIO outputs  */                          \
  LED1_DDR |= LED1_BV;                                           \
  HAL_TURN_OFF_LED1();  /* Add: Not using NWK_LED */             \
...
  /* configure tristates */                                      \
  P0INP |= PUSH2_BV;                                             \
                                                                 \
  /* Add: RS485 direction pin P1.3 setup */                      \
  P1SEL &= ~0x08;       /* Add: P1.3 manually control */         \
  P1DIR |= 0x08;        /* Add: P1.3 output */                   \
  P1_3 = 0;             /* Add: P1.3 default low (receive) */    \
                                                                 \
  /* setup RF frontend if necessary */                           \
...
```

### HAL\Target\CC2530ZNP\Drivers\hal_uart.c

```
#include "hal_board_cfg.h"
#include "hal_defs.h"
#include "hal_types.h"
#include "hal_uart.h"
#include "OnBoard.h"    // Add this line for MicroWait()
```

### HAL\Target\CC2530ZNP\Drivers\hal_uart.c -> _hal_uart_isr.c -> halUartTxIsr

Transmitting to host = H  Receiving from host = L

Add some code (according to [this article](https://e2e.ti.com/support/wireless-connectivity/zigbee-and-thread/f/158/t/131050?How-to-change-rs232-to-RS485-in-ZStack-CC2530-2-3-0-1-4-0)). **Pay attention to the lines with comment "Add: ..."**

### HAL\Target\CC2530ZNP\Drivers\hal_uart.c -> _hal_uart_isr.c

```
/*********************************************************************
 * LOCAL VARIABLES
 */

static uartISRCfg_t isrCfg;
static uint8 isSending; // Add: for signalling if whole packet is finished
```


### HAL\Target\CC2530ZNP\Drivers\hal_uart.c -> _hal_uart_isr.c -> HalUARTWriteISR

```
uint16 HalUARTWriteISR(uint8 *buf, uint16 len)
{
  uint16 cnt;
  
  // Enforce all or none.
  if (HalUARTTxAvailISR() < len)
  {
    return 0;
  }
  
  U1CSR &= ~0x40;     // Add: Disable RX
  P1_3 = 1;     // Add: 485 Transmitting mode (RUN_LED = 1)
  isSending = 1;        // Add: Set sending flag
  MicroWait(6000);      // Add: Wait for some time
  
  for (cnt = 0; cnt < len; cnt++)
  {
    isrCfg.txBuf[isrCfg.txTail] = *buf++;
    isrCfg.txMT = 0;

    if (isrCfg.txTail >= HAL_UART_ISR_TX_MAX-1)
    {
      isrCfg.txTail = 0;
    }
    else
    {
      isrCfg.txTail++;
    }

    // Keep re-enabling ISR as it might be keeping up with this loop due to other ints.
    IEN2 |= UTXxIE;  
  }
  
  isSending = 0;        // Add: Clear sending flag

  return cnt;
}
```

### HAL\Target\CC2530ZNP\Drivers\hal_uart.c -> _hal_uart_isr.c -> halUartTxIsr

```
#if (HAL_UART_ISR == 1)
HAL_ISR_FUNCTION( halUart0TxIsr, UTX0_VECTOR )
#else
HAL_ISR_FUNCTION( halUart1TxIsr, UTX1_VECTOR )
#endif
{
  if (isrCfg.txHead == isrCfg.txTail)
  {
    IEN2 &= ~UTXxIE;
    isrCfg.txMT = 1;
    
    if(!isSending)      // Only after HalUARTWriteISR() finished writing
    {
      MicroWait(6000);    // Add: Wait for some time
      U1CSR |= 0x40;      // Add: Enable RX
      P1_3 = 0;   // Add: 485 Receive mode (RUN_LED = 0)
    }
  }
  else
  {
    U1CSR &= ~0x40;     // Add: Keep RX disabled
    P1_3 = 1;     // Add: Keep 485 in transmitting mode (RUN_LED = 1)

    UTXxIF = 0;
    UxDBUF = isrCfg.txBuf[isrCfg.txHead++];

    if (isrCfg.txHead >= HAL_UART_ISR_TX_MAX)
    {
      isrCfg.txHead = 0;
    }
  }
}
```

## 5. Some change to LED and button config (not really effecting)

See diff file
