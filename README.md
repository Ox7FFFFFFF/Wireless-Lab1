# Wireless-Lab1
* 實驗環境
    * 硬體版本 : Raspberry Pi 3 Model B+
    * 作業系統 : Raspbian 2018-06-27
    * LoRa : SX1276晶片
* 實驗目標
    * 使用ABP模式傳輸資料
    * 並用MQTT取得資料後解碼
###### tags: `Wireless`

## :star: 實驗結果
1. 跑完ABP傳輸模式的流程，傳輸資料時分別使用`Confirm Data`和`Unconfirm Data`，截圖樹莓派上的結果，應該會有一個是有ACK一個是沒有ACK的。
2. 傳輸資料後用MQTT查看，並用base64解密截圖。
3. **以組為單位(一個人上傳即可)**，將這些截圖結果貼成一個word檔案，命名為`wireless-lab1-groupxx.docx`，並上傳到LMS作業區。


## APB模式送出 (send_ttn.py)
這份程式主要修改來自 [jeroennijhof](https://github.com/jeroennijhof/LoRaWAN) 的程式碼
* 將[實驗程式碼](https://github.com/Ox7FFFFFFF/Wireless-Lab1)載下來
    ```shell
    $ git clone https://github.com/Ox7FFFFFFF/Wireless-Lab1
    ```

    * 程式資料夾
    <i class="fa fa-folder-open"></i> LoRaWAN - 與LoRaWAN相關的程式
    <i class="fa fa-folder-open"></i> SX127x - 控制晶片的SPI程式
    <i class="fa fa-file-text"></i> send_ttn.py - 資料送出 (主程式)
    <i class="fa fa-file-text"></i> config.json - Activated [DevAddr, NwkSKey, AppSKey]配置

    * 程式流程
    ```flow
    st=>start: Start
    tx=>operation: MODE.TX
    send=>operation: send message
    tx_done=>operation: TX_DONE
    rx=>operation: MODE.RXSINGLE
    read=>operation: Read downlink payload
    save=>operation: Save fCnt to config.json
    cond=>condition: MIC check vaild
    rx_done=>operation: RX_DONE
    drop=>opertion: pass message
    success=>operation: Success
    fail=>operation: Fail
    e=>end: End
    
    st->tx->send->tx_done->tx_done->rx->rx_done->read->cond
    cond(yes)->success->save
    cond(no)->fail->save
    save->e
    ```

1. 將devAddr,nwkSKey,appSKey填入`config.json`中
    ```json
    {
        "appskey": "dcb80cced0527753276d8e49a11e546e",
        "devaddr": "07cdbe7a",
        "fCnt": 0,
        "nwskey": "0e73623166e5caa0e82b0360c1bcfc62"
    }
    ```

3. 調整`send_ttn.py`主要參數
    * Frequency 
    * Spreading Factor
    * Bandwidth
    ``` python=114
    # Setup
    lora.set_mode(MODE.SLEEP)
    lora.set_dio_mapping([1,0,0,0,0,0])
    lora.set_freq(AS923.FREQ1)
    lora.set_spreading_factor(SF.SF7)
    lora.set_bw(BW.BW125)
    lora.set_pa_config(pa_select=1)
    lora.set_pa_config(max_power=0x0F, output_power=0x0E)
    lora.set_sync_word(0x34)
    lora.set_rx_crc(True)
    ```

4. fCnt 要隨著傳輸次數增加，Server端收到小於目前記錄的fCnt將會丟棄封包不處理。
    ```python=56
        def send(self):
            global fCnt
            lorawan = LoRaWAN.new(nwskey, appskey)
            message = "HELLO WORLD!"
            # 資料打包,fCnt+1
            lorawan.create(MHDR.CONF_DATA_UP, {'devaddr': devaddr, 
            'fcnt': fCnt, 'data': list(map(ord, message)) })
            print("fCnt: ",fCnt)
            print("Send Message: ",message)
            fCnt = fCnt+1
            self.write_payload(lorawan.to_raw())
            self.set_mode(MODE.TX)
    ```
5. 當切換到`MODE.TX`時,送出uplink之後會跳到`on_tx_done`
    ```python=21
        def on_tx_done(self):
            # 紀錄送出的時間
            global TX_TIMESTAMP
            TX_TIMESTAMP = datetime.datetime.now().timestamp()
            print("TxDone\n")
            # 切換到rx
            self.set_mode(MODE.STDBY)
            self.clear_irq_flags(TxDone=1)
            self.set_mode(MODE.SLEEP)
            self.set_dio_mapping([0,0,0,0,0,0])
            self.set_invert_iq(1)
            self.reset_ptr_rx()
            sleep(1)
            self.set_mode(MODE.RXSINGLE)
    ```
6. 當接收到downlink時，會跳到`on_rx_done`
    ```python=
        def on_rx_done(self):
            print("RxDone")
            self.clear_irq_flags(RxDone=1)

            # 讀取 payload
            payload = self.read_payload(nocheck=True)
            lorawan = LoRaWAN.new(nwskey, appskey)
            lorawan.read(payload)
            print("get mic: ",lorawan.get_mic())
            print("compute mic: ",lorawan.compute_mic())
            print("valid mic: ",lorawan.valid_mic())
            # 檢查mic
            if lorawan.valid_mic():
                print("ACK: ",lorawan.get_mac_payload().get_fhdr().get_fctrl()>>5&0x01)
                print("direction: ",lorawan.get_direction())
                print("devaddr: ",''.join(format(x, '02x') for x in lorawan.get_devaddr()))
                write_config()
            else:
                print("Wrong MIC")
            sys.exit(0)
    ```
7. 執行`send_ttn.py`
    ```shell
    $ python3 send_ttn.py
    ```
    ![](https://i.imgur.com/KlXRgCd.png)
* MIC(Message Integrity Check):
    * 用來確認這個資料是不是自己的
* ACK(Acknowledge):
    * 因為傳送的資料格式是`MHDR.CONF_DATA_UP`，所以Downlink中會有ACK回應
* Direction:
    * Uplink : 0
    * Downlink : 1
* devaddr
    * 也可以確認一下devaddr看是不是自己的，採用[Little-Endian](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F)，比對的時候要倒過來看

## 使用MQTT查看傳輸到Server的資料
1. 請先安裝 [MQTT BOX](http://workswithweb.com/html/mqttbox/installing_apps.html#install_on_windows)
2. 新增 MQTT Client
![](https://i.imgur.com/TkaruZV.png)
3. 輸入 MQTT server 資訊
![](https://i.imgur.com/WYFFKcx.png)
* MQTT Client Name : `隨便取`
* Protocol : `mqtts/tls`
* Username : `engineer`
* Host : `mqtt.hscc.csie.ncu.edu.tw:1883`
* SSL/TLS Certificate Type : `CA signed server ceritificate`
* Password : `nculoraserver`
4. 新增 Subscriber，並按下subscribe
![](https://i.imgur.com/tLToON5.png)
* 主要格式為`application/[applicationID]/device/[devEUI]/rx`
    * applicationID : 5
    * devEUI : 就是分配到的EUI
    * rx : server端收到的
5. 送出訊息後查看，`data`使用base64加密，所以要查看原始資料需要用base64解密
![](https://i.imgur.com/229s4g4.png)
6. [base64解密](https://www.base64decode.org/)

![](https://i.imgur.com/amPBVso.png)





