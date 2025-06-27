# 操舵差分メモ

1. loop関数からUDPの送信部分を削除

```diff
-#ifdef ARDUINO_UNOR4_WIFI
-    //----- UDP 送信（100 ms 間隔） -----
-    static unsigned long t0 = 0;
-    if (millis() - t0 >= UDP_INTERVAL)
-    {
-        t0 = millis();
-
-        if (WiFi.status() == WL_CONNECTED)
-        {
-            ControlData controlData;
-            controlData.rudder = a0;
-            controlData.elevator = a1;
-            controlData.trim = trimVal;
-
-            // 計測のIPアドレスが不定なので、ブロードキャストとして送信
-            IPAddress localIP = WiFi.localIP();
-            IPAddress subnet = WiFi.localIP();
-            IPAddress measurementIP(
-                localIP[0] | (~subnet[0]),
-                localIP[1] | (~subnet[1]),
-                localIP[2] | (~subnet[2]),
-                localIP[3] | (~subnet[3]));
-
-            wifiUdp.beginPacket(measurementIP, UDP_PORT);
-            wifiUdp.write((uint8_t *)(&controlData), sizeof(ControlData));
-            wifiUdp.endPacket();
-        }
-    }
-#endif
```
