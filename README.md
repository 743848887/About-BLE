
##蓝牙中心设计模式 －－流程
1. 建立中心角色

2. 扫描外设（discover）

3. 连接外设(connect)

4. 扫描外设中的服务和特征(discover)

    - 4.1 获取外设的services

    - 4.2 获取外设的Characteristics,获取Characteristics的值，获取Characteristics的Descriptor和Descriptor的值

5. 与外设做数据交互(explore and interact)

6. 订阅Characteristic的通知

7. 断开连接(disconnect)

####步骤

- 1、生成一个中心管理者和一个保存外接设备数据数组
    CBCentralManager *manager;
    NSMutableArray *peripheralArrs;
         
        queue   设置为nil默认在在主线程
       
        option  
       
         NSString * const CBCentralManagerOptionShowPowerAlertKey
         对应一个NSNumber类型的bool值，用于设置是否在关闭蓝牙时弹出用户提示
      
         NSString * const CBCentralManagerOptionRestoreIdentifierKey 
         对应一个NSString对象，设置管理中心的标识符ID
       
         self.manager  =[[CBCentralManager alloc]initWithDelegate:self queue:nil options:nil];
       

- 2、扫描周围设备

          第一个为nil 默认扫描周围所有设备    活着扫描指定设备
          
          
          option   CBCentralManagerScanOptionAllowDuplicatesKey  设为YES 去除重覆扫描，只产生一个事件
          
          NO 扫描所有设备，并合并成一个事件。
               
          [self.manger scanForPeripheralsWithServices:
          @[UARTPeripheral.uartServiceUUID] options:
                   @{CBCentralManagerScanOptionAllowDuplicatesKey: [NSNumber numberWithBool:NO]}];


- 3、查到外设后，停止扫描，连接设备
     
          @param advertisementData 是外设发送的广播数据
          @param RSSI              信号强度
 
         - (void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)peripheral advertisementData:(NSDictionary<NSString *, id> *)advertisementData RSSI:(NSNumber *)RSSI {
         
         
          NSLog(@"Did discover peripheral %@", peripheral.name);
          
          [self.manger stopScan];---停止扫描
    
           self.peripheral = peripheral;
    
           self.peripheral.delegate = self;
    
          连接设备
          
          CBConnectPeripheralOptionNotifyOnDisconnectionKey  是否显示外接设备断开的警告
          
          
         [self.manger connectPeripheral:peripheral options:@{CBConnectPeripheralOptionNotifyOnDisconnectionKey: [NSNumber numberWithBool:YES]}];
           
        }



-  4.1获取外设的services       
            
         //连接外设成功，开始发现服务
        - (void)centralManager:(CBCentralManager *)central didConnectPeripheral: (CBPeripheral *)peripheral {
       
              NSLog(@"Did connect peripheral %@", peripheral.name);

        
              //扫描外设Services，成功后会进入方法：
              -(void)peripheral:(CBPeripheral *)peripheral didDiscoverServices:(NSError *)error{
              
            [p            eripheral discoverServices:nil];

        }
       
        
        //扫描到Services
        -(void)peripheral:(CBPeripheral *)peripheral didDiscoverServices:(NSError *)error{
            //  NSLog(@">>>扫描到服务：%@",peripheral.services);
            if (error)
            {
                NSLog(@">>>Discovered services for %@ with error: %@", peripheral.name, [error localizedDescription]);
                return;
            }

            for (CBService *service in peripheral.services) {
                             NSLog(@"%@",service.UUID);
                             //扫描每个service的Characteristics，扫描到后会进入方法： -(void)peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(NSError *)error
                             [peripheral discoverCharacteristics:nil forService:service];
                         }

        }
         
         
         
- 4.2获取外设的Characteristics,获取Characteristics的值，获取Characteristics的Descriptor和Descriptor的值

       //扫描到Characteristics
     -(void)peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(NSError *)error{
         if (error)
         {
             NSLog(@"error Discovered characteristics for %@ with error: %@", service.UUID, [error localizedDescription]);
             return;
         }

         for (CBCharacteristic *characteristic in service.characteristics)
         {
             NSLog(@"service:%@ 的 Characteristic: %@",service.UUID,characteristic.UUID);
         }

         //获取Characteristic的值，读到数据会进入方法：-(void)peripheral:(CBPeripheral *)peripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
         for (CBCharacteristic *characteristic in service.characteristics){
             {
                 [peripheral readValueForCharacteristic:characteristic];
             }
         }

         //搜索Characteristic的Descriptors，读到数据会进入方法：-(void)peripheral:(CBPeripheral *)peripheral didDiscoverDescriptorsForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
         for (CBCharacteristic *characteristic in service.characteristics){
             [peripheral discoverDescriptorsForCharacteristic:characteristic];
         }


        }

        //获取的charateristic的值
         -(void)peripheral:(CBPeripheral *)peripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error{
        //打印出characteristic的UUID和值
        //!注意，value的类型是NSData，具体开发时，会根据外设协议制定的方式去解析数据
        NSLog(@"characteristic uuid:%@  value:%@",characteristic.UUID,characteristic.value);

        }

         //搜索到Characteristic的Descriptors
          -(void)peripheral:(CBPeripheral *)peripheral didDiscoverDescriptorsForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error{

        //打印出Characteristic和他的Descriptors
         NSLog(@"characteristic uuid:%@",characteristic.UUID);
        for (CBDescriptor *d in characteristic.descriptors) {
            NSLog(@"Descriptor uuid:%@",d.UUID);
        }

         }
           //获取到Descriptors的值
         -(void)peripheral:(CBPeripheral *)peripheral didUpdateValueForDescriptor:(CBDescriptor *)descriptor error:(NSError *)error{
        //打印出DescriptorsUUID 和value
        //这个descriptor都是对于characteristic的描述，一般都是字符串，所以这里我们转换成字符串去解析
        NSLog(@"characteristic uuid:%@  value:%@",[NSString stringWithFormat:@"%@",descriptor.UUID],descriptor.value);
        
        }


- 5 把数据写到Characteristic中

             //写数据
    -(void)writeCharacteristic:(CBPeripheral *)peripheral
                characteristic:(CBCharacteristic *)characteristic
                         value:(NSData *)value{

        //打印出 characteristic 的权限，可以看到有很多种，这是一个NS_OPTIONS，就是可以同时用于好几个值，常见的有read，write，notify，indicate，知知道这几个基本就够用了，前连个是读写权限，后两个都是通知，两种不同的通知方式。
        /*
         typedef NS_OPTIONS(NSUInteger, CBCharacteristicProperties) {
         CBCharacteristicPropertyBroadcast												= 0x01,
         CBCharacteristicPropertyRead													= 0x02,
         CBCharacteristicPropertyWriteWithoutResponse									= 0x04,
         CBCharacteristicPropertyWrite													= 0x08,
         CBCharacteristicPropertyNotify													= 0x10,
         CBCharacteristicPropertyIndicate												= 0x20,
         CBCharacteristicPropertyAuthenticatedSignedWrites								= 0x40,
         CBCharacteristicPropertyExtendedProperties										= 0x80,
         CBCharacteristicPropertyNotifyEncryptionRequired NS_ENUM_AVAILABLE(NA, 6_0)		= 0x100,
         CBCharacteristicPropertyIndicateEncryptionRequired NS_ENUM_AVAILABLE(NA, 6_0)	= 0x200
         };

         */
        NSLog(@"%lu", (unsigned long)characteristic.properties);


        //只有 characteristic.properties 有write的权限才可以写
        if(characteristic.properties & CBCharacteristicPropertyWrite){
            /*
                最好一个type参数可以为CBCharacteristicWriteWithResponse或type:CBCharacteristicWriteWithResponse,区别是是否会有反馈
            */
            [peripheral writeValue:value forCharacteristic:characteristic type:CBCharacteristicWriteWithResponse];
        }else{
            NSLog(@"该字段不可写！");
        }


        }

         
- 6 订阅Characteristic的通知  
         
         
         
             //设置通知
         -(void)notifyCharacteristic:(CBPeripheral *)peripheral
                characteristic:(CBCharacteristic *)characteristic{
        //设置通知，数据通知会进入：didUpdateValueForCharacteristic方法
        [peripheral setNotifyValue:YES forCharacteristic:characteristic];

        }

           //取消通知
          -(void)cancelNotifyCharacteristic:(CBPeripheral *)peripheral
                 characteristic:(CBCharacteristic *)characteristic{

         [peripheral setNotifyValue:NO forCharacteristic:characteristic];
        
        
         }
         

- 7 断开连接(disconnect)

           
         //停止扫描并断开连接
         -(void)disconnectPeripheral:(CBCentralManager *)centralManager
                     peripheral:(CBPeripheral *)peripheral{
              //停止扫描
              [centralManager stopScan];
              //断开连接
              [centralManager cancelPeripheralConnection:peripheral];
        } 

         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
         
