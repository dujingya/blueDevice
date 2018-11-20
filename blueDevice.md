#使用mpvue 开发小程序过程中 简单介绍一下微信小程序蓝牙连接过程
#在蓝牙连接的过程中部分api需要加定时器延时1秒到2秒左右再执行，原因为何不知道，小程序有这样的要求
#1.首先是要初始化蓝牙：openBluetoothAdapter()
```js
if (wx.openBluetoothAdapter) {
    wx.openBluetoothAdapter({
        success: function(res) {
            /* 获取本机的蓝牙状态 */
            setTimeout(() => {
                getBluetoothAdapterState()
            }, 1000)
        },
        fail: function(err) {
          // 初始化失败
        }
    })
} else {
   
}
```

#2.检测本机蓝牙是否可用：
#  要在上述的初始化蓝牙成功之后回调里调用
```js
getBluetoothAdapterState() {
    var that = this;
    that.toastTitle = '检查蓝牙状态'
    wx.getBluetoothAdapterState({
        success: function(res) {
           startBluetoothDevicesDiscovery()
        },
        fail(res) {
            console.log(res)
        }
    })
}
```

#3. 开始搜索蓝牙设备：
```js
startBluetoothDevicesDiscovery() {
    var that = this;
    setTimeout(() => {
        wx.startBluetoothDevicesDiscovery({
            success: function(res) {
                /* 获取蓝牙设备列表 */
                that.getBluetoothDevices()
            },
            fail(res) {
            }
        })
    }, 1000)
}
```
 
#4. 获取搜索到的蓝牙设备列表
# /* that.deviceName 是获取到的蓝牙设备的名称， 因为蓝牙设备在安卓和苹果手机上搜到的蓝牙地址显示是不一样的，所以根据设备名称匹配蓝牙*/
```js
getBluetoothDevices() {
    var that = this;
    setTimeout(() => {
        wx.getBluetoothDevices({
            services: [],
            allowDuplicatesKey: false,
            interval: 0,
            success: function(res) {
                if (res.devices.length > 0) {
                    if (JSON.stringify(res.devices).indexOf(that.deviceName) !== -1) {
                        for (let i = 0; i < res.devices.length; i++) {
                            if (that.deviceName === res.devices[i].name) {
                                /* 根据指定的蓝牙设备名称匹配到deviceId */
                                that.deviceId = that.devices[i].deviceId;
                                setTimeout(() => {
                                    that.connectTO();
                                }, 2000);
                            };
                        };
                    } else {
                    }
                } else {
                }
            },
            fail(res) {
                console.log(res, '获取蓝牙设备列表失败=====')
            }
        })
    }, 2000)
},
```
#5.连接蓝牙 
# 匹配到的蓝牙设备ID 发送连接蓝牙的请求， 连接成功之后 应该断开蓝牙搜索的api，然后去获取所连接蓝牙设备的service服务
```js
connectTO() {
    wx.createBLEConnection({
        deviceId: deviceId,
        success: function(res) {
            that.connectedDeviceId = deviceId;
            /* 4.获取连接设备的service服务 */
            that.getBLEDeviceServices();
            wx.stopBluetoothDevicesDiscovery({
                success: function(res) {
                    console.log(res, '停止搜索')
                },
                fail(res) {
                }
            })
        },
        fail: function(res) {
        }
    })
}
```
#6. 获取蓝牙设备的service服务,获取的serviceId有多个要试着连接最终确定哪个是稳定版本的service 获取服务完后获取设备特征值
```js
getBLEDeviceServices() {
    setTimeout(() => {
        wx.getBLEDeviceServices({
            deviceId: that.connectedDeviceId,
            success: function(res) {
                that.services = res.services
                /* 获取连接设备的所有特征值 */
                that.getBLEDeviceCharacteristics()
            },
            fail: (res) => {
            }
        })
    }, 2000)
},
```
#7.获取蓝牙设备特征值 
# 获取到的特征值有多个，最后要用的事能读，能写，能监听的那个值的uuid作为特征值id，
```js
 getBLEDeviceCharacteristics() {
            setTimeout(() => {
                wx.getBLEDeviceCharacteristics({
                    deviceId: connectedDeviceId,
                    serviceId: services[2].uuid,
                    success: function(res) {
                        for (var i = 0; i < res.characteristics.length; i++) {
                            if ((res.characteristics[i].properties.notify || res.characteristics[i].properties.indicate) &&
                                (res.characteristics[i].properties.read && res.characteristics[i].properties.write)) {
                                console.log(res.characteristics[i].uuid, '蓝牙特征值 ==========')
                                /* 获取蓝牙特征值 */
                                that.notifyCharacteristicsId = res.characteristics[i].uuid
                                // 启用低功耗蓝牙设备特征值变化时的 notify 功能
                                that.notifyBLECharacteristicValueChange()
                            }
                        }
                    },
                    fail: function(res) {
                    }
                })
            }, 1000)
        },
```
#8.启动notify 蓝牙监听功能 然后使用 wx.onBLECharacteristicValueChange用来监听蓝牙设备传递数据
#接收到的数据和发送的数据必须是二级制数据， 页面展示的时候需要进行转换
```js
        notifyBLECharacteristicValueChange() { // 启用低功耗蓝牙设备特征值变化时的 notify 功能
            var that = this;
            console.log('6.启用低功耗蓝牙设备特征值变化时的 notify 功能')
            wx.notifyBLECharacteristicValueChange({
                state: true,
                deviceId: that.connectedDeviceId,
                serviceId: that.notifyServicweId,
                characteristicId: that.notifyCharacteristicsId,
                complete(res) {
                    /*用来监听手机蓝牙设备的数据变化*/
                    wx.onBLECharacteristicValueChange(function(res) {
                        /**/
                        that.balanceData += that.buf2string(res.value)
                        that.hexstr += that.receiveData(res.value)
                    })
                },
                fail(res) {
                    console.log(res, '启用低功耗蓝牙设备监听失败')
                    that.measuringTip(res)
                }
            })
        },
        
        /*转换成需要的格式*/
         buf2string(buffer) {
                    var arr = Array.prototype.map.call(new Uint8Array(buffer), x => x)
                    return arr.map((char, i) => {
                        return String.fromCharCode(char);
                    }).join('');
        },
          receiveData(buf) {
                    return this.hexCharCodeToStr(this.ab2hex(buf))
          },
          /*转成二进制*/
           ab2hex (buffer) {
              var hexArr = Array.prototype.map.call(
                  new Uint8Array(buffer), function (bit) {
                      return ('00' + bit.toString(16)).slice(-2)
                  }
              )
              return hexArr.join('')
          },
          /*转成可展会的文字*/
          hexCharCodeToStr(hexCharCodeStr) {
              var trimedStr = hexCharCodeStr.trim();
              var rawStr = trimedStr.substr(0, 2).toLowerCase() === '0x' ? trimedStr.substr(2) : trimedStr;
              var len = rawStr.length;
              var curCharCode;
              var resultStr = [];
              for (var i = 0; i < len; i = i + 2) {
                  curCharCode = parseInt(rawStr.substr(i, 2), 16);
                  resultStr.push(String.fromCharCode(curCharCode));
              }
              return resultStr.join('');
          },
```
# 向蓝牙设备发送数据
```js
sendData(str) {
    let that = this;
    let dataBuffer = new ArrayBuffer(str.length)
    let dataView = new DataView(dataBuffer)
    for (var i = 0; i < str.length; i++) {
        dataView.setUint8(i, str.charAt(i).charCodeAt())
    }
    let dataHex = that.ab2hex(dataBuffer);
    this.writeDatas = that.hexCharCodeToStr(dataHex);
    wx.writeBLECharacteristicValue({
        deviceId: that.connectedDeviceId,
        serviceId: that.notifyServicweId,
        characteristicId: that.notifyCharacteristicsId,
        value: dataBuffer,
        success: function (res) {
            console.log('发送的数据：' + that.writeDatas)
            console.log('message发送成功')
        },
        fail: function (res) {
        },
        complete: function (res) {
        }
    })
    },
```
# 当不需要连接蓝牙了后就要关闭蓝牙，并关闭蓝牙模块
```js
 // 断开设备连接
closeConnect() {
    if (that.connectedDeviceId) {
        wx.closeBLEConnection({
            deviceId: that.connectedDeviceId,
            success: function(res) {
                that.closeBluetoothAdapter()
            },
            fail(res) {
            }
        })
    } else {
        that.closeBluetoothAdapter()
    }
},
// 关闭蓝牙模块
closeBluetoothAdapter() {
    wx.closeBluetoothAdapter({
        success: function(res) {
        },
        fail: function(err) {
        }
    })
},
```
#在向蓝牙设备传递数据和接收数据的过程中，并未使用到read的API 不知道有没有潜在的问题，目前线上运行为发现任何的问题
#今天的蓝牙使用心得到此结束，谢谢
