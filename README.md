# Smarttrashcan-STM32

> 智慧垃圾桶 STM32 專案

## 專案簡介

本專案示範如何以 **STM32** 微控制器（主要以 STM32F4 系列為例）結合 **HC-SR04 超音波感測器**、**SG90 伺服馬達** 與 **LED**，實現一個「智慧垃圾桶」基本功能：  
1. 使用超音波感測器量測前方距離，當距離小於設定閾值（例如 20 cm）時觸發開蓋動作。  
2. 伺服馬達根據 PWM 信號自動打開或關閉垃圾桶蓋。  
3. LED 作為狀態指示（開蓋：LED 常亮或閃爍；關蓋：LED 熄滅）。  
4. （可選）透過藍牙或 UART 顯示／遠端監控量測距離與開關狀態。

整個系統以 **STM32CubeMX + HAL** 驅動程式開發，並示範如何使用定時器（TIM）產生 PWM、GPIO + EXTI 處理 Echo 中斷、以及基本的 UART 通訊。專案範例預設以 **STM32F401**（或 **STM32F429/Nucleo-F429ZI**）為開發板，但您可以輕鬆地移植到其他 STM32F4/STM32F1 系列 MCU。

---

## 目錄

- [硬體需求](#硬體需求)  
- [連接說明](#連接說明)  
- [軟體需求](#軟體需求)  
- [專案結構](#專案結構)  
- [建置與燒錄](#建置與燒錄)  
- [核心程式流程示意](#核心程式流程示意)  
- [操作說明](#操作說明)  
- [注意事項](#注意事項)  
- [License](#license)  

---

## 硬體需求

1. **MCU 開發板**  
   - 例如：STM32F401RE (NUCLEO-F401RE)，或 STM32F429ZI (NUCLEO-F429ZI)。  
   - 板上需內建 ST-Link，以便透過 USB 直接燒錄程式。  

2. **HC-SR04 超音波感測器**  
   - Trig 腳：輸入 10 µs 短脈衝以觸發量測。  
   - Echo 腳：輸出回波脈衝，對應距離計算。  
   - Echo 訊號為 5 V，必須做電平轉換或分壓到 3.3 V，才能安全接入 STM32 的 GPIO。  

3. **SG90 伺服馬達**  
   - 使用 5 V 外部電源驅動，由 STM32 的 TIM PWM 腳輸出控制訊號。  
   - PWM 頻率約 50 Hz，佔空比範圍約 0.5 ms（0°）–2.5 ms（180°）。  

4. **LED（狀態指示燈）**  
   - 接到 3.3 V GPIO，可透過定時器或軟體迴圈實作閃爍效果（例如 1 秒亮、2 秒暗）。  

5. **（可選）藍牙模組**  
   - JDY-31、HC-05、HM-10 等 3.3 V TTL 藍牙模組，透過 UART 與 STM32 通訊。  
   - 若模組電平為 5 V TTL，需額外做邏輯電平轉換。  

6. **電源供應**  
   - STM32 板：USB 供電 → 板內穩壓 3.3 V。  
   - HC-SR04、SG90 與 LED 外掛：可由 STM32 板上的 5 V 腳位供電，並將 GND 共地。  
   - 若伺服馬達耗流較大，建議使用獨立 5 V 電源並共地，以免 USB 供電不足。

---

## 連接說明

以下範例以 **STM32F4 Nucleo-F401RE**（或 NUCLEO-F429ZI）板為例，Arduino 風格接腳與 HAL 名稱對照如下：

| 功能             | 板上針腳 (Arduino / HAL) | 模組腳位    | 說明                                                             |
|------------------|-------------------------|-------------|------------------------------------------------------------------|
| HC-SR04 Trig     | D2 (PA 10)              | Trig        | GPIO 輸出，高電位維持 10 μs 觸發量測                                 |
| HC-SR04 Echo     | D3 (PB 3)               | Echo        | GPIO 輸入，用 EXTI / TIM Input Capture 計算 Echo 高電位持續時間    |
| SG90 PWM (0°–90°)| D6 (PA 8)               | Control     | TIM1_CH1 PWM，頻率 50 Hz，輸出 0.5 – 2.5 ms 佔空比驅動伺服          |
| LED（狀態指示）  | D13 (PA 5)              | LED         | 板載綠色 LED，可用 GPIO high/low 或用 TIM 產生閃爍                |
| 藍牙 TX (RX STM) | D0 (PA 3)               | RX (STM)    | USART2_RX（3.3 V TTL），接模組 TX                                   |
| 藍牙 RX (TX STM) | D1 (PA 2)               | TX (STM)    | USART2_TX（3.3 V TTL），接模組 RX                                   |
| 電源 3.3 V       | 3.3 V                   | VCC (STM)   | STM32 板既有電源輸出                                                    |
| 電源 5 V         | 5 V                     | VCC (外插)  | HC-SR04、SG90、LED 外掛                                           |
| 地線 (GND)       | GND                     | GND         | 需將所有模組與 MCU 共地                                           |

> **特別提醒**  
> 1. Echo 與藍牙模組 TX 輸出若為 5 V TTL，務必加電阻分壓 (例如 10 kΩ + 20 kΩ) 或使用電平轉換器降至 3.3 V。  
> 2. SG90 在啟動瞬間耗流較大，建議增加去耦電容（如 100 μF）並考慮使用獨立 5 V 電源以穩定電壓。  

---

## 軟體需求

- **開發環境**  
  - [STM32CubeIDE](https://www.st.com/product/STM32CubeIDE)（建議 v1.9 以上）  
  - [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)（若要自行修改 .ioc 設定）  
  - GNU Arm Embedded Toolchain（若使用命令列方式）  

- **HAL 驅動版本**  
  - STM32CubeF4 HAL 驅動程式庫 (v1.25.0 以上)  
  - CMSIS Core  

- **版本控制**  
  - Git（若要在本機以 CLI 管理，並推送至 GitHub）  

---

## 專案結構

以下範例結構示意了常見的 STM32CubeMX 專案檔案與資料夾位置，實際結構依您匯入／生成後可能略有差異：

