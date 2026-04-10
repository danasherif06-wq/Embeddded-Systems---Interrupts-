[10/10][Feedback]
- Excellent


---
Dana Khalil: g00098584
Omar Matar: b00099010
Farah Tawalbeh: g00099768
---

# COE411L-00 Report 3
### Introduction
This report presents the implementation and validation of interrupt-driven systems using the STM32 Nucleo-L476RG. The objective of this lab was to understand how external and internal interrupts operate, how Interrupt Service Routines (ISRs) interact with the main program, and how event-driven design improves embedded system efficiency.

Interrupts allow a microcontroller to respond immediately to hardware events without continuously polling inputs. When an interrupt occurs, the processor temporarily suspends normal execution, executes a callback function (ISR), and then resumes the main program. This approach reduces CPU load, improves responsiveness, and enables efficient real-time behavior.

Two interrupt mechanisms were implemented:

- External Interrupt (EXTI) using a Ball Switch
A digital ball switch connected to PA0 was configured to trigger an interrupt on a falling edge. Upon motion detection, the ISR incremented a counter and set a flag. The main loop then handled the event by generating a short buzzer alert and applying software debouncing to prevent multiple triggers caused by mechanical bouncing.

- Internal Interrupt using the ADC Analog Watchdog with an LDR
An LDR connected to ADC channel 6 was monitored using the Analog Watchdog feature. Upper and lower thresholds were configured so that an interrupt was generated only when the light intensity exceeded the defined range. The ISR updated an event counter and set a flag, while the main loop read the ADC value, activated the buzzer, and transmitted the measurement and event count over USART2.

Both experiments emphasized proper embedded design practice: ISRs remain short and set flags, while the main loop performs hardware interaction and communication. The lab provided practical experience with NVIC configuration, EXTI lines, ADC watchdog setup, callback functions, debouncing, and UART-based debugging.

### Description
## System 1: External Interrupt (Ball Switch + Active Buzzer)
A ball switch was connected to PA0 and configured as EXTI0 with falling-edge detection. When the board is tilted or shaken, the ball switch changes state, triggering an interrupt.
Inside the interrupt callback:
- A counter variable is incremented
- An event flag is set
The main loop checks this flag. When set, it activates the buzzer for 50 ms and clears the flag. Software debouncing was applied to reduce multiple triggers caused by mechanical bouncing.

## System 2: Internal Interrupt (LDR + ADC Analog Watchdog)
An LDR voltage divider was connected to PA1 (ADC Channel 6). The ADC Analog Watchdog was configured to generate an interrupt when the ADC reading moves outside a defined threshold window.
If the ADC value exceeds the upper threshold or drops below the lower threshold, an interrupt occurs.
Inside the ADC callback:
- A counter variable is incremented
- An event flag is set
  
The main loop responds by:
- Reading the ADC value
- Activating the buzzer
- Printing the ADC value and interrupt count to USART2
If no interrupt occurs, the ADC value is periodically read and printed every 100 ms.


### Circuit Diagram

## Experiment 1 (Ball Switch + Buzzer)
<img width="479" height="455" alt="image" src="https://github.com/user-attachments/assets/571dc4d3-d99f-4276-b0b1-6c3017403931" />

## Experiment 2 (LDR + Buzzer + UART)

<img width="509" height="520" alt="image" src="https://github.com/user-attachments/assets/0f1910e4-a207-4ec2-b095-4a4e568707ab" />

### Key Code Segments

## Exercise 1: External Interrupt (EXTI) Implementation

```
volatile uint32_t g_ball_count = 0;   // total triggers
volatile uint8_t  g_ball_event = 0;   // event flag set by ISR
```
Two global variables are declared and marked as volatile:

g_ball_count: Stores the total number of times the ball switch interrupt has been triggered.
g_ball_event: Acts as a flag to notify the main loop that an interrupt event has occurred.

The volatile keyword is required because these variables are modified inside an Interrupt Service Routine (ISR). Since interrupts occur asynchronously, the compiler must not optimize access to these variables.

```
void Buzzer_Beep(uint32_t ms) {
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_4, GPIO_PIN_SET);   // Buzzer ON
  HAL_Delay(ms);
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_4, GPIO_PIN_RESET); // Buzzer OFF
}
```
The Buzzer_Beep function generates a short audible signal using the active buzzer connected to pin PB4.

Function operation:
- Sets PB4 HIGH to turn the buzzer ON.
- Delays for the specified number of milliseconds.
- Sets PB4 LOW to turn the buzzer OFF.

```
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
  if (GPIO_Pin == GPIO_PIN_0) {      // Ball switch on PA0
    g_ball_count++;
    g_ball_event = 1;
  }
}
```

This function is automatically executed when an external interrupt occurs on a configured GPIO pin. In this experiment, PA0 is configured as EXTI line 0 with falling edge detection.

When the ball switch changes state (due to motion or tilt):
- The EXTI0 interrupt is triggered.
- The HAL library calls HAL_GPIO_EXTI_Callback().

The function checks whether the interrupt originated from GPIO pin 0.

If true:
- The total trigger counter (g_ball_count) is incremented.
- The event flag (g_ball_event) is set to 1.

## Exercise 1: Main Loop
```
int last = 0;

while (1)
{
    if (g_ball_event)
    {
        int start = HAL_GetTick();

        if ((start - last) > 150)
        {
            g_ball_event = 0;
            Buzzer_Beep(50);
            last = start;
        }
    }
}
```

1) Event Detection Using a Flag
Inside the while(1) loop, the program checks whether g_ball_event is set:
If g_ball_event == 1, this indicates that an external interrupt occurred (ball switch detected motion).
The main loop then proceeds to handle the event (rather than doing the work inside the ISR).

2) Time-Based Debouncing
Ball switches are mechanical devices and can “bounce,” producing multiple rapid interrupts from a single shake. To prevent multiple buzzer activations for one physical movement, the code implements a debounce delay window using the system tick timer.

HAL_GetTick() returns the current elapsed time in milliseconds since startup.
last stores the timestamp of the last accepted event.
start stores the current time when a new event is detected.

The code accepts a new event only if: start - last > 150 ms
This means any triggers that occur within 150 ms of the previous accepted trigger are ignored.

3) Handling the Valid Event

When a trigger passes the debounce condition:

The event flag is cleared: g_ball_event = 0;
A short beep is generated: Buzzer_Beep(50);
The last event time is updated: last = start;

As a result, the buzzer beeps once per valid shake, while suppressing extra triggers caused by contact bouncing.

## Exercise 2: Internal Interrupt (ADC Analog Watchdog)
```
volatile uint32_t g_ldr_count = 0;
volatile uint8_t  g_ldr_event = 0;
volatile uint16_t g_adc_last = 0;
char msg[64];
```
g_ldr_count: Counts how many times the Analog Watchdog interrupt has occurred (i.e., how many out-of-window events were detected).
g_ldr_event: A flag set by the ADC ISR to notify the main loop that an out-of-window event happened.
g_adc_last: Stores the most recent ADC reading (useful for logging/printing the value that caused the event).
msg[64]: A character buffer used to format UART messages (e.g., via sprintf) before transmitting.

```
uint32_t Read_ADC_Channel(uint32_t channel)
{
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel = channel;
    sConfig.Rank = ADC_REGULAR_RANK_1;
    sConfig.SamplingTime = ADC_SAMPLETIME_47CYCLES_5;

    HAL_ADC_ConfigChannel(&hadc1, &sConfig);
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);

    return HAL_ADC_GetValue(&hadc1);
}
```
This function reads the analog voltage from the specified ADC channel and returns the digital ADC result.

1. Configure the ADC channel
2. sConfig.Channel = channel selects which ADC input to read (e.g., ADC_CHANNEL_6 for PA1 on ADC1 IN6).
3. Rank 1 ensures this channel is the first and only conversion in the regular sequence.
4. Sampling time is set to 47.5 cycles.
5. HAL_ADC_Start(&hadc1) begins the conversion process.
6. HAL_ADC_PollForConversion(..., HAL_MAX_DELAY) blocks until the conversion completes.
7. HAL_ADC_GetValue(&hadc1) returns a 12-bit result in the range 0–4095 (for a 12-bit ADC).

```
void HAL_ADC_LevelOutOfWindowCallback(ADC_HandleTypeDef* hadc) {
  if (hadc->Instance == ADC1) {
    g_ldr_count++;
    g_ldr_event = 1; // flag for main loop
  }
}
```
This callback is executed automatically when the ADC Analog Watchdog detects the ADC value is outside the configured thresholds (too bright or too dark).

1. Confirms the interrupt came from ADC1 (hadc->Instance == ADC1).
2. Increments the event counter g_ldr_count.
3. Sets the event flag g_ldr_event = 1.

```
void Buzzer_Beep(uint32_t ms) {
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_4, GPIO_PIN_SET); // buzzer ON
  HAL_Delay(ms);
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_4, GPIO_PIN_RESET); // buzzer OFF
}
```
This function provides an audible indication whenever a valid event is handled in the main loop.

1. Set PB4 HIGH → buzzer ON
2. Delay for ms milliseconds
3. Set PB4 LOW → buzzer OFF

Because this uses HAL_Delay, it should be called in the main loop (not inside the ISR) to avoid blocking interrupt execution.

## Exercise 2: Main Logic

The main program continuously executes inside the infinite while(1) loop. The loop is responsible for handling interrupt events and providing continuous monitoring of the LDR sensor.

Inside the loop, the program first checks whether the interrupt flag g_ldr_event has been set:
```
if (g_ldr_event)
```
This flag is set inside the ISR (HAL_ADC_LevelOutOfWindowCallback) whenever the ADC value goes outside the predefined threshold window (too bright or too dark).

When an event is detected: Reset the event flag
```
g_ldr_event = 0;
```
This prevents the same event from being handled multiple times.

Read the current ADC value
```
g_adc_last = Read_ADC_Channel(ADC_CHANNEL_6);
```
The program reads ADC channel 6 (connected to PA1, the LDR input) and stores the value in g_adc_last.

Generate audible feedback
```
Buzzer_Beep(50);
```
A short 50 ms beep is produced to indicate that the light level has exceeded the defined threshold.

Transmit event information via UART

```
sprintf(msg, "LDR=%u  Count=%lu\r\n", g_adc_last, g_ldr_count);
HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
```
The program prints:

- The ADC value that triggered the event
- The total number of watchdog events (g_ldr_count)

This provides both audible and visual confirmation of significant light changes.

Normal Monitoring Case

If no interrupt event has occurred:
```
else
```
The system continues to monitor the LDR value:

Read ADC channel 6
```
uint16_t adc = Read_ADC_Channel(ADC_CHANNEL_6);
```
Print the current light level
```
sprintf(msg, "LDR=%u\r\n", adc);
HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
```
This allows continuous observation of the light intensity even when it remains within the safe threshold window.

Timing Control

At the end of each loop iteration:
```
HAL_Delay(100);
```
A 100 ms delay is introduced to:

- Prevent excessive UART flooding
- Maintain stable sampling intervals
- Reduce unnecessary CPU usage
- Improve readability of the serial output
  
### System Photo
## Exercise 1
<img width="468" height="426" alt="image" src="https://github.com/user-attachments/assets/65eaaff1-3c67-4cf3-9496-9f0f41a6640a" />

## Exercise 2
<img width="481" height="360" alt="image" src="https://github.com/user-attachments/assets/397ee7b4-8340-4556-ac38-534a0df52e01" />

## Screenshots

## Exercise 1 IOC configurations
<img width="627" height="326" alt="image" src="https://github.com/user-attachments/assets/a13355b4-1884-4d44-a242-f3dcbaa9c6b9" />
<img width="623" height="350" alt="image" src="https://github.com/user-attachments/assets/ca3a924f-0660-4058-b0c5-df1ab5dc3ebc" />
<img width="629" height="369" alt="image" src="https://github.com/user-attachments/assets/ff76b9ad-71ed-4427-8ce9-32fd4652f281" />

## Exercise 2 IOC configurations
<img width="626" height="440" alt="image" src="https://github.com/user-attachments/assets/212122d8-3127-490a-9c65-3767982b9e8f" />
<img width="625" height="348" alt="image" src="https://github.com/user-attachments/assets/fcda8b2f-b43a-411c-ade7-e658e2e0d6d1" />
<img width="634" height="358" alt="image" src="https://github.com/user-attachments/assets/b9f10a58-1a3f-4a88-90ef-88b9cc37ee43" />

## Exercise 2 UART 
<img width="616" height="322" alt="image" src="https://github.com/user-attachments/assets/912bccc1-aafa-43e5-8e6c-c15598d5c58e" />

### Conclusion

This lab successfully demonstrated the implementation of both external and internal interrupt mechanisms on the STM32 Nucleo-L476RG. The EXTI experiment validated event-driven digital input handling, including correct NVIC configuration, ISR execution, and software debouncing. The ADC Analog Watchdog experiment demonstrated hardware-based threshold monitoring for analog signals, enabling efficient detection of significant light changes without continuous polling.

The implementation reinforced key embedded system principles:
- Interrupt-driven design improves responsiveness and efficiency.
- ISRs must be short, non-blocking, and limited to flag setting.
- Shared variables between ISR and main code require the volatile qualifier.
- Peripheral actions and communication belong in the main loop.
Overall, the lab strengthened understanding of interrupt architecture, callback mechanisms, and event-driven programming—core concepts in real-time embedded system design.
