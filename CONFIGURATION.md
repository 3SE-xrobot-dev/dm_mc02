# Current board configuration

The source of truth is `dm_mc02.ioc`; the matching HAL initialization is in
`Core/Src/main.c` and `Core/Src/stm32h7xx_hal_msp.c`.

- HSE 24 MHz; CPU 480 MHz; HCLK 240 MHz; APB 120 MHz.
- SPI2: BMI088, mode 3, 8 bits, 3.75 MHz. PC0 is ACC_CS, PC3_C is
  GYRO_CS; both idle high. PE10/PE12 are rising-edge ACC_INT/GYRO_INT.
  PC3_C needs the SYSCFG analog switch closed.
- SPI6: WS2812, TX-only, PA7 MOSI, PB3 SCK. HSE 24 MHz / 4 gives
  6 MHz SCK for the Y reference driver's 8-SPI-bit encoding per LED bit
  (0x60 / 0x78). PA6 is no longer SPI6 MISO.
- TIM3 CH4 on PB1 (HEATER_PWM): PWM mode 1, active high, prescaler 239,
  period 9999, 100 Hz. TIM12 CH2 on PB15 (BUZZER_PWM): PWM mode 1,
  active high, prescaler 239, period 249, 4 kHz initially. Both start
  at 0% duty. Use `Board_Heater_SetDutyPermyriad(0..10000)` and
  `Board_Buzzer_SetTone(20..20000, 0..100)`; a zero frequency or
  volume turns the buzzer off. These provide PWM output only; BMI088
  temperature sampling and heater feedback control are application logic.
- UART5: DR16 receiver, 100000 baud, 9-bit word including even parity,
  2 stop bits, RX-only. PD2 is DR16_RX. PC12 remains assigned as UART5_TX
  in the pin map but the UART transmitter is disabled.
- FDCAN1 uses FIFO0; FDCAN2/3 use FIFO1. No standard filter elements are
  allocated. The global filters accept unmatched standard and extended
  frames, including remote frames. Initialize a receive consumer and start
  each FDCAN peripheral in application code when ready.
- DMA follows the Q stream assignment for enabled peripherals: ADC1
  DMA1 Stream0 (circular), UART5 RX Stream1, SPI2 RX/TX Stream2/3,
  UART7 RX/TX Stream4/5, USART2 RX/TX DMA2 Stream0/1,
  USART3 RX/TX Stream2/3, USART10 RX/TX Stream4/5. All enabled
  external interrupts use priority 5. USART1 uses DMA1 Stream6/7 at
  921600 baud as a general-purpose serial port; its application role
  is not assigned.

DMA1/2 cannot access DTCM, where ordinary globals and the heap currently
reside. Declare DMA buffers with `DMA_BUFFER` from `Core/Inc/main.h`;
the linker places them in the non-cacheable 32 KB D2 SRAM region. Example:
`DMA_BUFFER uint8_t imu_rx[64];`. This section is NOLOAD, so initialize
its contents before TX. Keep all DMA buffers within 32 KB. Regenerating
the linker script or MPU code with CubeMX requires preserving the
`.dma_buffer` section and the MPU region for 0x30000000.

The CAN peripheral is configured to accept all IDs but is not started
automatically. Start it after installing the application's receive path;
otherwise its FIFO can fill while no code drains it.
