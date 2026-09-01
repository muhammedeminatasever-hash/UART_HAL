
İki Nucleo geliştirme kartı arasında USART3 üzerinden basit bir haberleşme örneği. STM32F446RE her 2 saniyede bir `"50\\r\\n"` mesajını gönderir ve başarılı her gönderimde kendi LD2 LED'ini toggle eder. STM32F072RB bu mesajı interrupt tabanlı olarak alır, `"50"` ile başlayan bir satır tespit ettiğinde LD2'yi 500 ms yakar.

---

## Donanım

| Parametre | STM32F446RE (Verici) | STM32F072RB (Alıcı) |
| :--- | :--- | :--- |
| **Periferi** | USART3 | USART3 |
| **TX Pini** | PB10 | PC4 *(kullanılmıyor)* |
| **RX Pini** | PC5 *(kullanılmıyor)* | PC5 |
| **LED** | PA5 (LD2) | PA5 (LD2) |
| **Baud Rate** | 115200, 8N1 | 115200, 8N1 |

---

## Bağlantı Şeması

```text
F446RE PB10 (USART3_TX)  ───────►  F072RB PC5  (USART3_RX)
F446RE GND                ───────  F072RB GND
