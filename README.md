# 操舵差分メモ

元のコードのバックアップを取り、次の操作を実行

1. loop関数からUDPの送信部分を「削除」

```cpp
#ifdef ARDUINO_UNOR4_WIFI
    //----- UDP 送信（100 ms 間隔） -----
    static unsigned long t0 = 0;
    if (millis() - t0 >= UDP_INTERVAL)
    {
        t0 = millis();

        if (WiFi.status() == WL_CONNECTED)
        {
            ControlData controlData;
            controlData.rudder = a0;
            controlData.elevator = a1;
            controlData.trim = trimVal;

            // 計測のIPアドレスが不定なので、ブロードキャストとして送信
            IPAddress localIP = WiFi.localIP();
            IPAddress subnet = WiFi.localIP();
            IPAddress measurementIP(
                localIP[0] | (~subnet[0]),
                localIP[1] | (~subnet[1]),
                localIP[2] | (~subnet[2]),
                localIP[3] | (~subnet[3]));

            wifiUdp.beginPacket(measurementIP, UDP_PORT);
            wifiUdp.write((uint8_t *)(&controlData), sizeof(ControlData));
            wifiUdp.endPacket();
        }
    }
#endif
```

2. グローバル変数部分に次を「追加」

```cpp
#include <Arduino_FreeRTOS.h>

TaskHandle_t loop_task, udp_task;
void loop_thread_func(void *pvParameters);
void udp_thread_func(void *pvParameters);
```

3. setup関数の「末尾に」次を「追加」

```cpp
void setup() {
...
↓ここから

    auto const rc_loop = xTaskCreate(
        loop_thread_func, static_cast<const char *>("Loop Thread"), 4096, nullptr, 2, &loop_task);
    auto const rc_udp = xTaskCreate(
        udp_thread_func, static_cast<const char *>("UDP Thread"), 4096, nullptr, 1, &udp_task);

    vTaskStartScheduler();
    for (;;)
        ;
↑ここまでを追加
}
```

4. ソースコードの末尾に次を「追加」

```cpp
void loop_thread_func(void *pvParameters)
{
    for (;;)
    {
        loop();
        delay(10);
    }
}

void udp_thread_func(void *pvParameters)
{
    //----- UDP 送信（100 ms 間隔） -----
    for (;;)
    {
        if (WiFi.status() == WL_CONNECTED)
        {
            ControlData controlData;
            controlData.rudder = a0;
            controlData.elevator = a1;
            controlData.trim = trimVal;

            // 計測のIPアドレスが固定でないため、ブロードキャストとして送信
            IPAddress localIP = WiFi.localIP();
            IPAddress subnet = WiFi.localIP();
            IPAddress measurementIP(
                localIP[0] | (~subnet[0]),
                localIP[1] | (~subnet[1]),
                localIP[2] | (~subnet[2]),
                localIP[3] | (~subnet[3]));

            wifiUdp.beginPacket(measurementIP, UDP_PORT);
            wifiUdp.write((uint8_t *)(&controlData), sizeof(ControlData));
            wifiUdp.endPacket();
        }
        delay(100);
    }
}
```
