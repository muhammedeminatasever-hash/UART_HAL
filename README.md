# STM32 Nucleo UART Haberleşme — Verici / Alıcı

İki Nucleo geliştirme kartı arasında USART3 üzerinden basit bir haberleşme örneği. **STM32F446RE** her 2 saniyede bir `"50\r\n"` mesajını gönderir ve başarılı her gönderimde kendi LD2 LED'ini toggle eder. **STM32F072RB** bu mesajı interrupt tabanlı olarak alır, `"50"` ile başlayan bir satır tespit ettiğinde LD2'yi 500 ms yakar.

## Donanım

| | STM32F446RE (Verici) | STM32F072RB (Alıcı) |
|---|---|---|
| Periferi | USART3 | USART3 |
| TX pini | PB10 | PC5 *(kullanılmıyor)* |
| RX pini | PC5 *(kullanılmıyor)* | PC10 |
| LED | PA5 (LD2) | PA5 (LD2) |
| Baud rate | 115200, 8N1 | 115200, 8N1 |

### Bağlantı şeması

```
F446RE PB10 (USART3_TX)  ───────►  F072RB PC10 (USART3_RX)
F446RE GND                ───────  F072RB GND
```

Sadece tek yönlü haberleşme kullanıldığından PC5 hatları bağlanmasa da olur. İki yönlü haberleşme isteniyorsa `F446RE PC5 (RX)` ile `F072RB PC5 (TX)` da çapraz bağlanabilir.

## Çalışma mantığı

### Verici (F446RE)

- Ana döngüde 2 saniyede bir `HAL_UART_Transmit()` ile `"50\r\n"` gönderilir.
- Gönderim `HAL_OK` dönerse LD2 toggle edilir; böylece iletim başarısı LED üzerinden gözle takip edilebilir.

### Alıcı (F072RB)

- `HAL_UART_Receive_IT()` ile tek byte'lık kesme tabanlı alım başlatılır.
- Her gelen byte `HAL_UART_RxCpltCallback()` içinde bir satır tamponuna yazılır.
- `\n` (0x0A) görülünce satır tamamlanmış sayılır; tampon `"50"` ile başlıyorsa `msg_received` bayrağı set edilir.
- `\r` karakteri yok sayılır.
- Ana döngü bayrağı görünce LD2'yi 500 ms yakıp söndürür (kesme içinde `HAL_Delay` kullanılmaz).
- Satır sonu karakterine (`\n`) dayalı senkronizasyon sayesinde kayıp/gürültülü byte olsa bile alıcı bir sonraki satırla otomatik olarak yeniden hizalanır — sabit uzunluklu pencere kullanan yaklaşımlardaki kilitlenme riski ortadan kalkar. Tampon taşması durumunda da otomatik sıfırlama yapılır.

## Proje yapısı

```
├── main_F446RE_transmitter_FIXED.c   # Verici kart main.c
└── main_F072RB_receiver_FIXED.c      # Alıcı kart main.c
```

Her iki dosya da STM32CubeIDE ile oluşturulmuş standart proje iskeletindeki `main.c` dosyasının yerine konacak şekilde hazırlanmıştır (`USER CODE BEGIN/END` blokları korunmuştur).

## Bilinen düzeltmeler

Bu sürüm, önceki bir taslakta bulunan iki sorunu giderir:

- **F072RB tarafında derleme hatası:** `MX_USART3_UART_Init()` ile `huart3` init ediliyor, ancak alım ve callback çağrıları tanımsız bir `huart2` değişkenine referans veriyordu (`undefined reference to huart2`). Tüm çağrılar `huart3`'e ve callback kontrolü `USART3` instance'ına düzeltildi.
- **F446RE tarafında yanıltıcı yorum:** Dosya başlığında "USART2 TX, PA2" yazıyordu, ancak kod ve CubeMX ayarı fiilen USART3 (PB10/PC5) kullanıyordu. Yorum gerçek pin atamasıyla tutarlı hale getirildi; kodun işlevsel kısmında değişiklik yapılmadı.

## Kurulum

1. Her iki proje için de STM32CubeIDE'de ilgili Nucleo kartına (F446RE / F072RB) uygun bir proje oluşturun ya da mevcut `.ioc` dosyanızın USART3 ayarlarının yukarıdaki tabloyla eşleştiğini doğrulayın (Baud: 115200, 8N1, TX/RX pinleri).
2. İlgili `main.c` dosyasını bu depodaki karşılığıyla değiştirin.
3. Derleyip her iki karta da yükleyin.
4. Kartları yukarıdaki şemaya göre TX↔RX ve GND-GND olacak şekilde bağlayın.
5. F446RE üzerindeki LD2'nin 2 saniyede bir toggle ettiğini, F072RB üzerindeki LD2'nin ise her mesaj alımında 500 ms yandığını gözlemleyin.

## Sorun giderme

- **F072RB'de LED hiç yanmıyor:** TX/RX çapraz bağlantısını kontrol edin (TX→RX, RX→TX), GND hattının ortak olduğundan emin olun.
- **Rastgele/seyrek yanıp sönme:** Baud rate uyuşmazlığı veya gürültülü kablo bağlantısı olabilir; her iki tarafta da 115200 8N1 ayarını doğrulayın.
- **Derleme hatası (`huart2` tanımsız):** Eski/düzeltilmemiş F072RB `main.c` dosyasını kullanıyor olabilirsiniz — bu depodaki düzeltilmiş sürümü kullanın.
