
> 在Android4.4之后的版本中，Android在硬件中支持内置计步传感器。例如微信运动，支付宝运动等常用软件都是直接调用了Android中的Sensor传感器服务，从而获取到每日的步数。
要完成计步传感器的调用，需要了解**Sensor**，**SensorManager**，**SensorEventListener**，**SensorEvent**四个类。

### Sensor
**Sensor**即传感器，该类的路径为*android.hardware.Sensor*。可以使用**SensorManager**的`getSensorList(int)`来获取可用的传感器列表。
在本文中获取计步传感器是通过`SensorManager.getDefaultSensor(int type)`来获取的.用的比较多的有3种，

##### 加速度传感器 *Sensor.TYPE_ACCELEROMETER* 计步方式
这种方式是有开源算法根据加速度传感器进行计算步数；

优点：只要有加速度传感器的设备都可以使用，相对来说可以使用的设备比较多

缺点：步数的准确性取决于算法且算法比较难优化；需要后台保活**Service**否者不能计步；计步算法比较费电，部分手机锁屏不能计步

##### 计步传感器 *Sensor.TYPE_STEP_COUNTER* 计步方式
这种类型的传感器返回是从最后一次系统重启到当前时间的所有步数。以一个浮点类型的值返回，只有在手机重启系统的时候才会重设为0.同时返回一个时间戳，记录最后一次返回步数的时间。
因为这个计步器是硬件上实现的，所以预计耗电量非常少。如果你想要长期计步的话，注册的传感器不要注销，它会在后台持续计步，即使当前应用处于休眠状态。当应用唤醒时返回当前计步总数。
应用必需在这个传感器上注册，激活计步器，否者计步器是不会计步的。

优点：硬件计步准确性高，电耗小；只需要注册不用启用后台service，就可以自动注册

缺点：只支持Android 4.4系统以上的部分手机，手机系统重启的时候计步器会清零；不能返回步数明细，只能返回最后一次重启到当前时间的步数；

##### 检测传感器 *Sensor.TYPE_STEP_DETECTOR*
这种类型的传感器是用户每迈出一步，此传感器就会触发一个事件。对于每个用户步伐，此传感器提供一个返回值为 1.0 的事件和一个指示此步伐发生时间的时间戳。
当用户在行走时，会产生一个加速度上的变化，从而出触发此传感器事件的发生。注意此传感器只能检测到单个有效的步伐，获取单个步伐的有效数据，
如果需要统计一段时间内的步伐总数，则需要使用下面的 `TYPE_STEP_COUNTER` 传感器

优点：无需后台service，注册即可，每走一步触发一次

缺点：不记录总数，有误差，应用处于休眠状态时，无法计步

以上是3种在计步器中比较常用的计步方式。但是由于`TYPE_ACCELEROMETER`算法比较难优化，有需要后台启动service，所以放弃这种计步方式。下面主要讨论一下
`TYPE_STEP_DETECTOR` 和 `TYPE_STEP_COUNTER` 这两种计步传感器这两者的比较、用法以及优劣

注意：`TYPE_STEP_DETECTOR` 和 `TYPE_STEP_COUNTER` 这两种计步传感器都需要依赖硬件，可以使用`PackageManager的hasSystemFeature(String name)`方法来检测硬件是否支持该传感器，
这两个传感器对应的常量为`FEATURE_SENSOR_STEP_DETECTOR` 和 `FEATURE_SENSOR_STEP_COUNTER` 。

		getPackageManager().hasSystemFeature(PackageManager.FEATURE_SENSOR_STEP_COUNTER)

		getPackageManager().hasSystemFeature(PackageManager.FEATURE_SENSOR_STEP_DETECTOR)
		
注意：这两种计步器提供的结果并非总是相同。与来自 `TYPE_STEP_DETECTOR` 的事件相比，`TYPE_STEP_COUNTER` 事件的发生延迟时间更长，但这是因为 `TYPE_STEP_COUNTER` 算法会进行较多的处理以消除误报。
因此，`TYPE_STEP_COUNTER` 在传输事件时可能较为缓慢，但其结果应更为准确。


### SensorManager
**SensorManager** 是一个系统服务，如果需要获取某个传感器**Sensor**，则需要通过该服务来获取。如果需要获取该系统服务，可以通过`Context.getSystemService()`方法来获取：

    sensorManager= (SensorManager) getSystemService(SENSOR_SERVICE);

注意：当不需要使用传感器或者应用程序进入后台时。需要手动关闭传感器。如果不及时关闭这些传感器是非常耗电的。当关闭屏幕时，系统也不会自动帮你关闭传感器；

### SensorEventListener
**SensorEventListener**顾名思义就是一个传感器的事件监听器，当系统服务SensorManager中注册传感器是需要提供一个传感器事件监听器来获取传感器发生的事件，
该监听器只有两个需要实现的方法，即`onAccuracyChanged(Sensor sensor, int accuracy)`和`onSensorChanged(SensorEvent event)`

    onAccuracyChanged(Sensor sensor, int accuracy) //当注册的传感器的精确度发生变化时会触发该方法

    onSensorChanged(SensorEvent event) //当注册的传感器发生新事件时会触发该方法

### SensorEvent
SensorEvent 即传感器事件，包括了传感器的名字，类型，事件发生时间戳，精确度和传感器数据等信息。

> `accuracy` 精确度信息

> `sensor`  触发该事件的传感器

> `timestamp`  事件发生的时间戳，有19位，需要除以1000000才能获取到Unix时间戳

>  `values`  values是一个数组，对应不同的传感器类型，该数组的长度和内容都是不一样的，
例如可以是`values[0]`代表**x**轴，`values[1]`代表**y**轴，`values[2]`代表**z**轴等数据。
正常情况下，计步传感器只需要使用`values[0]`即可获取到步伐数据。




解决计步传感器的一些限制（系统重启清零，不能自动分天，部分手机进程杀死不能计步）

文献参考：

[Android计步模块优化（今日步数）](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/1027/8650.html)

[Android 计步传感器的实现](http://blog.csdn.net/u010144805/article/details/79096732)
