# BÁO CÁO TỔNG HỢP NGHIÊN CỨU
## Thiết kế và Xử lý Tín hiệu cho Ống nghe Y tế Điện tử sử dụng 2× INMP441 và ESP32-S3 N16R8

---

## Tóm tắt

Báo cáo này tổng hợp toàn bộ quá trình tìm hiểu tài liệu khoa học phục vụ đề tài NCKH "Ống nghe y tế điện tử", bao gồm: đặc điểm tần số của âm thanh cơ thể (tim/phổi), các phương pháp lọc nhiễu và tăng cường âm thanh, phân tích chuyên sâu về khả năng khai thác hệ 2 microphone trong thiết kế cụ thể, vấn đề ghép nối âm học giữa ống nghe và mic MEMS, các lựa chọn cảm biến thay thế, khả năng triển khai trên phần cứng ESP32-S3 N16R8, và pipeline xử lý tín hiệu được đề xuất cuối cùng.

---

## 1. Đặt vấn đề

Hệ thống nghiên cứu gồm: hộp cách âm bằng bìa formex có lót xốp, bên trong đặt 2 microphone MEMS INMP441 ở hai phía trái/phải, mỗi bên có một lỗ để đưa ống nghe y tế vào; vi điều khiển xử lý là ESP32-S3 N16R8, giao tiếp với mic qua chuẩn I2S. Mục tiêu là thu được âm thanh cơ thể (chủ yếu âm tim) có chất lượng tốt, giảm nhiễu môi trường, và triển khai được toàn bộ pipeline xử lý real-time trên chính vi điều khiển này.

Ba đặc điểm kỹ thuật chi phối gần như mọi lựa chọn thuật toán trong đề tài:
1. Tín hiệu mục tiêu có tần số rất thấp (gần hạ âm), trong khi khoảng cách giữa 2 mic lại rất nhỏ.
2. Cả hai mic đều đặt gần lỗ đưa ống nghe vào — khác với thiết kế "1 mic tín hiệu + 1 mic nhiễu tham chiếu" phổ biến trong y văn.
3. Nền tảng xử lý là một vi điều khiển (MCU), không phải máy tính hay DSP chuyên dụng.

---

## 2. Đặc điểm tần số của âm thanh ống nghe

| Thành phần | Dải tần số | Nguồn |
|---|---|---|
| Âm tim — tần số trội (dominant) | 50–150 Hz | [1] |
| Âm tim — tần số phụ (subdominant, đôi khi biên độ lớn hơn tần số trội) | 450–650 Hz | [1] |
| Âm tim cơ bản + tiếng thổi (murmur), dùng làm băng lọc phổ biến | 20–400 Hz | [2] |
| Các biến thể ngưỡng cắt Butterworth khác đã dùng trong y văn | 25–400Hz, 25–500Hz, 25–250Hz, 50–950Hz, 30–900Hz | [3] |
| Đỉnh phổ FFT quan sát thực nghiệm trên PCG (2 đỉnh) | 19–50Hz và 50–100Hz | [4] |
| Âm phổi (hô hấp), theo một số patent/nghiên cứu | 140–1600Hz hoặc 50–2500Hz (cao hơn hẳn âm tim) | — |

**Kết luận:** dải 20–150Hz người dùng đề xuất ban đầu là hợp lý và khớp với tần số trội của âm tim [1], nhưng nếu muốn giữ lại tiếng thổi tim (murmur) — vốn có giá trị chẩn đoán — nên mở rộng dải lọc tới 400Hz theo cách nhiều nghiên cứu phân loại PCG đã làm [2],[3].

---

## 3. Các phương pháp lọc nhiễu và tăng cường âm thanh

### 3.1. Lọc tần số (tiền xử lý)

Theo khảo sát tổng quan về phân tích âm tim bằng deep learning, bộ lọc thông cao thường dùng để loại nhiễu tần số rất thấp, còn bộ lọc thông dải (bandpass) được dùng phổ biến hơn cả vì loại được đồng thời cả nhiễu tần số thấp lẫn tần số cao; trong đó **Butterworth bandpass** đã được áp dụng thành công trong rất nhiều nghiên cứu với các bậc và ngưỡng cắt khác nhau [3]. Một nghiên cứu cụ thể dùng Butterworth bậc 4, ngưỡng cắt 20–400Hz, dựa trên cơ sở âm tim cơ bản và murmur nằm trong dải này [2].

### 3.2. Khử nhiễu nâng cao

- **Spectral subtraction / Wiener filter**: hiệu quả với nhiễu nền tương đối ổn định (stationary), nhưng dễ gây méo dạng "musical noise" nếu nhiễu thay đổi nhanh; chi phí tính toán cao hơn do cần FFT.
- **Wavelet denoising (DWT)**: xử lý tốt nhiễu không ổn định. Một nghiên cứu đã triển khai thành công thuật toán khử nhiễu wavelet thời gian thực để phát hiện QRS trên vi điều khiển **MSP430F169 16-bit** — yếu hơn ESP32-S3 rất nhiều — với ADC 12-bit, lấy mẫu 800Hz cho phát hiện QRS và 762Hz cho khử nhiễu, đạt độ chính xác và SNR ở mức chấp nhận được [5]. Điều này chứng minh DWT hoàn toàn khả thi trên phần cứng của đề tài.
- **LMS/NLMS/RLS/Kalman**: các bộ lọc thích nghi, phù hợp khi có tín hiệu tham chiếu rõ ràng (xem mục 4).

### 3.3. Công cụ tương ứng trong Audacity (dùng để thử nghiệm nhanh trước khi code)

| Audacity | Vai trò |
|---|---|
| High Pass / Low Pass Filter | Tương đương bước lọc tần số ở trên; kết hợp 2 filter = bandpass |
| Filter Curve EQ | Vẽ trực tiếp hình dạng bandpass hoặc notch tùy ý |
| Noise Reduction | Tương đương spectral subtraction, cần lấy mẫu nhiễu (Noise Profile) trước |
| Noise Gate | Cắt nhiễu nền trong khoảng lặng |
| Amplify / Normalize | Chuẩn hóa biên độ, loại DC offset |
| Compressor | Nén dải động, làm đều biên độ S1/S2 |

---

## 4. Phân tích hệ thống 2 microphone

### 4.1. Vì sao beamforming và GCC-PHAT không phù hợp

Tài liệu kinh điển "Microphone Arrays: A Tutorial" của Iain McCowan chỉ rõ: hiệu năng ở tần số thấp là điểm yếu cố hữu của các kỹ thuật beamforming thông thường, vì bước sóng lớn khiến sai lệch pha giữa các cảm biến đặt gần nhau trở nên không đáng kể, dẫn tới khả năng phân biệt hướng kém [6]. Một patent về mảng mic tính toán cụ thể: để beamforming hiệu quả xuống tới 20Hz, khoảng cách giữa các phần tử mic cần tới 17 mét — hoàn toàn phi thực tế để chế tạo [7]. Một patent khác về beamforming thích nghi xác nhận thêm: với khoảng cách mic rất nhỏ, các phương pháp beamforming thông thường đòi hỏi khuếch đại đáng kể ở tần số thấp (vì tín hiệu giữa các mic gần như giống hệt nhau), khiến SNR ở tần số thấp trở nên kém [8].

→ Với khoảng cách 2 mic chỉ vài cm trong hộp của đề tài và tín hiệu mục tiêu ở 20–150/400Hz, **beamforming/GCC-PHAT nên bị loại khỏi phạm vi nghiên cứu vì lý do vật lý**, không phải vì khó lập trình.

### 4.2. Vì sao kỹ thuật ANC "mic tham chiếu" kinh điển không khớp hoàn toàn

Các thiết kế ống nghe điện tử chống ồn trong y văn thường dùng kiến trúc: một mic nhận âm ống nghe qua cổng thông với đường khí, và một mic thứ hai nhận nhiễu môi trường qua cổng riêng, **được cách ly cơ học hoàn toàn khỏi đường khí đó** [9]. Một patent cũ hơn còn chỉ ra lý do vị trí 2 mic quan trọng đến mức nào: bất kỳ sự dịch chuyển vị trí nào giữa 2 cảm biến trong không gian cũng gây ra chênh lệch pha của nhiễu tới 2 mic thay đổi theo hướng nguồn nhiễu, làm giảm hiệu quả khử nhiễu tối đa có thể đạt được — vì vậy 2 cảm biến trong thiết kế ANC cần đặt càng gần nhau càng tốt [10].

Trong thiết kế của đề tài, **cả 2 mic đều đặt gần lỗ đưa ống nghe vào**, nghĩa là cả hai đều "thấy" tín hiệu ống nghe ở mức tương quan cao — không có mic nào đóng vai trò "chỉ nghe nhiễu" như kiến trúc chuẩn ở trên. Do đó:
- Trừ trực tiếp Mic1 − Mic2 có nguy cơ triệt tiêu cả tín hiệu mong muốn.
- NLMS kiểu "mic tham chiếu" cổ điển có thể hoạt động sai lệch vì giả định nền tảng của thuật toán không đúng với setup này.

### 4.3. Phương pháp phù hợp hơn: Averaging / Coherence-based 2-channel

Vì tín hiệu ống nghe tương quan cao ở cả 2 mic trong khi nhiễu điện tử của từng mic MEMS là độc lập, kỹ thuật **averaging 2 kênh** (hoặc nâng cao hơn là coherence-based weighting) phù hợp về mặt nguyên lý hơn so với ANC/beamforming, đồng thời có chi phí tính toán cực thấp.

### 4.4. So sánh LMS vs NLMS (nếu vẫn muốn thử nghiệm bổ sung)

Một nghiên cứu so sánh trực tiếp trên mảng 4 mic cho thấy NLMS cải thiện chất lượng giọng nói với mức khử nhiễu lên tới 13dB, trong khi LMS chỉ đạt khoảng 10dB [11] — nên nếu có thử ANC, ưu tiên NLMS hơn LMS thuần.

---

## 5. Vấn đề ghép nối âm học (acoustic coupling) và giải pháp

Một vấn đề thực nghiệm quan trọng phát sinh trong quá trình làm đề tài: âm thanh từ ống nghe đến mic INMP441 rất yếu hoặc không nghe thấy, do không tiếp xúc trực tiếp và không đủ kín khí.

**Cơ sở vật lý:** âm tần số thấp cần một khoang khí gần như kín để áp suất âm "dồn" được vào màng mic; chỉ một khe hở nhỏ cũng khiến hệ thống hoạt động như bộ lọc thông cao tự nhiên, cắt mất đúng dải tần thấp cần thu. Một thảo luận kỹ thuật thực tế xác nhận: dù dùng loại mic nào, "chỉ cần một khe khí nhỏ cũng khiến nó vô dụng" [12], và một trường hợp cụ thể dùng electret mic cho ống nghe điện tử có tín hiệu yếu hóa ra do một lỗ hở trên ống cao su — sau khi vá lại thì thu bình thường [13].

**Giải pháp đề xuất (ưu tiên theo thứ tự):**
1. Làm kín khí tuyệt đối tại điểm nối: dùng khớp nối vừa khít (in 3D hoặc silicone), ống co nhiệt, hoặc keo silicone/hot glue trám kín mọi khe hở.
2. Đảm bảo cổng âm của INMP441 hướng thẳng và áp sát đầu ra ống nghe; có thể đặt mic lồng ngay trong ống dẫn âm, bọc quanh bằng lớp foam mỏng "trong suốt về âm" vừa giữ kín khí vừa giảm rung cơ học — nguyên lý dùng trong một số patent ống nghe điện tử [14].
3. Sau khi đã kín khí, mới tính đến khuếch đại số (digital gain, tận dụng dữ liệu I2S 24-bit) hoặc thêm khoang cộng hưởng nhỏ.
4. Chỉ cân nhắc đổi sang cảm biến tiếp xúc (piezo) nếu phương án khí thực sự bế tắc, vì piezo thu qua tiếp xúc cơ học nên không phụ thuộc vào việc bịt kín khí — nhưng cần mạch đệm trở kháng cao và đổi hẳn cơ chế ghép nối.

---

## 6. Lựa chọn cảm biến (mic) thay thế — có cần đổi INMP441 không?

**Không nhất thiết.** INMP441 là mic MEMS kỹ thuật số — chính loại được nhiều nghiên cứu về cảm biến âm tim khuyến nghị dùng thay piezo, vì MEMS cho SNR tốt và mạch đơn giản hơn (I2S số, không cần mạch phân cực analog) [15]. Một nghiên cứu so sánh trực tiếp còn cho thấy mic MEMS và electret condenser mic (ECM) cho kết quả tương đồng cao khi đo âm cơ thể, và MEMS là lựa chọn thay thế khả thi về chi phí cho ECM [16].

| Phương án | Độ nhạy tần số thấp | Mạch phụ trợ | Độ phức tạp khi đổi |
|---|---|---|---|
| Electret Condenser Mic (ECM) | Tốt nếu màng lớn, vẫn cần ghép khí kín | Cần mạch phân cực + op-amp, dùng ADC thay vì I2S | Trung bình |
| MEMS I2S khác (vd ICS-43432) | Tương đương INMP441 | Không cần, vẫn I2S | Thấp (gần như thay thế trực tiếp) |
| Piezo / Contact transducer | Rất tốt, không phụ thuộc khí | Cần mạch đệm trở kháng cao + ADC | Cao (đổi cả cơ chế ghép nối) |

Một điểm cần lưu ý về piezo: mạch thông thường không thiết kế cho phần tử điện dung cao như piezo nên dễ gây méo âm "tinny" nếu không có mạch đệm phù hợp — điều này đã được ghi nhận cả trong thảo luận kỹ thuật thực tế lẫn nghiên cứu học thuật [17],[18].

**Khuyến nghị:** trước khi đổi mic, nên kiểm tra bằng cách nghe trực tiếp bằng tai người ở đúng điểm ghép nối — nếu tai người cũng khó nghe thấy gì, vấn đề chắc chắn nằm ở phần cơ khí/ghép nối chứ không phải ở mic.

---

## 7. Khả năng triển khai trên ESP32-S3 N16R8

Thư viện chính thức **ESP-DSP** của Espressif cung cấp sẵn: nhân ma trận, dot product, FFT, IIR, FIR, các phép toán vector, và cả **Kalman filter** — được tối ưu bằng assembly cho CPU ESP32 [19]. Có ví dụ thực tế trên bo mạch **ESP32-S3-BOX-Lite** dùng chức năng FFT của ESP-DSP để xử lý luồng âm thanh từ **hai microphone đồng thời** [20] — đúng use-case của đề tài.

- Band-pass IIR: rất nhẹ, chạy real-time dễ dàng.
- Wavelet denoising: đã chứng minh khả thi trên MCU yếu hơn ESP32-S3 nhiều lần [5].
- RLS: nặng nhất do cần nghịch đảo ma trận mỗi mẫu — xếp vào nhóm không ưu tiên.
- Beamforming: về code hoàn toàn chạy được trên ESP32, nhưng vô nghĩa về vật lý ở dải tần này (xem mục 4.1) nên không nên đầu tư công sức.

---

## 8. Pipeline xử lý tín hiệu đề xuất

```
2× INMP441 (I2S) → DC removal → Band-pass IIR Butterworth (20–200/400Hz, bậc 4)
   → Averaging 2 mic (tận dụng tín hiệu tương quan cao, nhiễu điện tử độc lập)
   → Wavelet denoising (DWT, khử nhiễu không ổn định còn lại)
   → Normalize / Compressor (tùy chọn)
   → Output
```

Pipeline này được chọn vì: (1) bám sát dải tần đã được xác nhận qua nhiều nghiên cứu PCG [1]-[3]; (2) khai thác đúng đặc điểm vật lý của thiết kế 2 mic (tương quan tín hiệu cao, không dùng beamforming/ANC sai giả định) [6]-[10]; (3) dùng các thuật toán đã được chứng minh chạy tốt trên phần cứng yếu hơn ESP32-S3 [5]; (4) có sẵn công cụ thư viện hỗ trợ trên ESP-DSP [19],[20].

---

## 9. Thiết kế thực nghiệm và chỉ số đánh giá

So sánh 3 bộ dữ liệu cùng điều kiện: **Raw** (không xử lý) / **Traditional** (chỉ band-pass tĩnh) / **Proposed** (pipeline đề xuất ở mục 8). Các chỉ số cần đo: SNR, SNR improvement (dB), RMS, phổ FFT và spectrogram (trực quan hóa phần năng lượng bị loại), độ méo tín hiệu (so sánh hình dạng S1/S2 trước/sau lọc), độ trễ xử lý, CPU/RAM usage trên ESP32-S3 (đo bằng cơ chế cycle count có sẵn trong ESP-DSP).

---

## 10. Kết luận

- Dải tần 20–150Hz là hợp lý, nên cân nhắc mở rộng tới 200–400Hz nếu muốn giữ murmur.
- Beamforming/GCC-PHAT nên loại khỏi hướng nghiên cứu vì lý do vật lý (bước sóng ≫ khoảng cách mic).
- Kiến trúc 2 mic của đề tài không khớp với mô hình ANC-mic-tham-chiếu kinh điển; nên ưu tiên averaging/coherence-based.
- Vấn đề tín hiệu yếu nhiều khả năng do hở khí tại điểm ghép nối, không phải do mic — nên kiểm tra và khắc phục phần cơ khí trước khi cân nhắc đổi cảm biến.
- Pipeline khuyến nghị: DC removal → Band-pass Butterworth → Averaging 2 mic → Wavelet denoising → Normalize, toàn bộ đều khả thi trên ESP32-S3 N16R8 nhờ thư viện ESP-DSP.
- Việc còn thiếu: dữ liệu thực nghiệm đo mức tương quan/độ trễ pha thực tế giữa 2 mic của chính thiết kế hộp — cần đo trước khi khẳng định chắc chắn phương pháp 2-kênh nào tối ưu nhất.

---

## Tài liệu tham khảo

[1] Frequency Extraction of Phonocardiogram Signal using Fourier Transform — Jurnal Elektro (ejournal.atmajaya.ac.id)

[2] Deep Learning Based Classification of Unsegmented Phonocardiogram Spectrograms Leveraging Transfer Learning — arXiv:2012.08406

[3] A Comprehensive Survey on Heart Sound Analysis in the Deep Learning Era — arXiv:2301.09362

[4] Heart Sound Segmentation Using Deep Learning Techniques — arXiv:2406.05653

[5] Optimization and implementation of the wavelet based algorithms for embedded biomedical signal processing — ComSIS Vol. 10, No. 1 (2013)

[6] Microphone Arrays: A Tutorial — Iain McCowan (2001)

[7] Band-limited beamforming microphone array — US Patent 10397697

[8] Signal processing methods and systems for adaptive beam forming — US Patent 12075217

[9] Electronic stethoscope device with noise cancellation — US Patent 11741931

[10] Noise-reducing stethoscope — US Patent 5492129A

[11] Comparison of LMS and NLMS algorithm with the using of 4 Linear Microphone Array for Speech Enhancement — European Journal of Engineering and Technology Research (2017)

[12] Best microphone to detect heartbeat — All About Circuits forum

[13] Sthetoscop digital using electret condenser mic (ECM) — All About Circuits forum

[14] Digital stethoscope and monitoring instrument — US Patent Application US20080013747A1

[15] Heart sound sensing through MEMS Microphone — ResearchGate

[16] Wearable technologies for joint health assessment — US Patent 11071494

[17] Electronic Stethoscope, Electret condenser mic distortion issues — Electro-Tech-Online forum

[18] A First Step Towards On-Device Monitoring of Body Sounds in the Wild — arXiv:2008.05370

[19] ESP-DSP — official DSP library, Espressif (GitHub: espressif/esp-dsp)

[20] ESP-DSP spectrum_box_lite example (ESP32-S3-BOX-Lite, FFT trên 2 mic) — components.espressif.com
