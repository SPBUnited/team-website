# Пример программы

Рассмотрим взаимодействие участника с программами [SSL-Vision](../software/programs.md#ssl-vision), [GrSim](../software/programs.md#grsim) и [Game Controller](../software/programs.md#ssl-game-controller). Для удобства, будем рассматривать уже готовый пример, созданный бразильской командой RobôCIn специально для новых команд в лиге. Ниже будет разбираться актуальный (на момент декабря 2024) код в [данном репозитории](https://www.github.com/robocin/ssl-client). Вы можете самостоятельно собрать проект по инструкции из описания репозитория и убедиться в том, что всё работает.

В файле [*src/main.cpp*](https://github.com/robocin/ssl-client/blob/master/src/main.cpp) реализована основная программа, она:

* получает от SSL-Vision/GrSim координаты мяча и роботов, а также геометрические размеры поля
* выводит их пользователю
* отправляет команды управления на синих роботов.

>По регламенту лиги, взаимодействие между программами в общей сети осуществляется с помощью протокола [Google Protobuf](https://github.com/protocolbuffers/protobuf). Стандартные адреса и порты, а также содержание передаваемых пакетов можно посмотреть [тут](https://ssl.robocup.org/league-software/#:~:text=Simulation%20Protocol.-,Standard%20Network%20Parameters,-Protocol).

Также вы можете ознакомиться с нашей реализацией - программой [LARCmaCS](../software/programs.md#larcmacs).

## Получение данных от [SSL-Vision](../software/programs.md#ssl-vision)

В данном репозитории за получение данных отвечает класс **RoboCupSSLClient**, функционал которого описан в файле [.../robocup_ssl_client.cpp](https://github.com/robocin/ssl-client/blob/master/include/ssl-client/net/robocup_ssl_client/robocup_ssl_client.cpp).

### Формат получаемых данных
<code>packet</code> (инициализируется на [строке 31](https://github.com/robocin/ssl-client/blob/master/src/main.cpp#L31)) - пакет, содержащий информацию о поле, полученный от SSL-Vision/GrSim. 
Далее программа проверяет, есть ли в этом пакете данные о захваченном с поля (*detection*, проверяется на [строке 42](https://github.com/robocin/ssl-client/blob/master/src/main.cpp#L42)) или геометрические размеры поля ([строки 101-143](https://github.com/robocin/ssl-client/blob/master/src/main.cpp#L101-L143)).
В первом случае, программа выводит информацию о работе SSL-Vision ([строки 47-59](https://github.com/robocin/ssl-client/blob/master/src/main.cpp#L47-L59)), обрабатывает данные о мяче ([строки 64-79](https://github.com/robocin/ssl-client/blob/master/src/main.cpp#L64-L79)) и о роботах ([строки 81-99](https://github.com/robocin/ssl-client/blob/master/src/main.cpp#L81-L99)).

Все передаваемые параметры *detection* можно найти в файле [.../messages_robocup_ssl_detection.proto](https://github.com/robocin/ssl-client/blob/master/include/ssl-client/protobuf-files/pb/proto/messages_robocup_ssl_detection.proto)

```
syntax = "proto2";

message SSL_DetectionBall {
  required float  confidence = 1;
  optional uint32 area       = 2;
  required float  x          = 3;
  required float  y          = 4;
  optional float  z          = 5;
  required float  pixel_x    = 6;
  required float  pixel_y    = 7;
}

message SSL_DetectionRobot {
  required float  confidence  =  1;
  optional uint32 robot_id    =  2;
  required float  x           =  3;
  required float  y           =  4;
  optional float  orientation =  5;
  required float  pixel_x     =  6;
  required float  pixel_y     =  7;
  optional float  height      =  8;
}

message SSL_DetectionFrame {
  required uint32             frame_number  = 1;
  required double             t_capture     = 2;
  required double             t_sent        = 3;
  required uint32             camera_id     = 4;
  repeated SSL_DetectionBall  balls         = 5;
  repeated SSL_DetectionRobot robots_yellow = 6;
  repeated SSL_DetectionRobot robots_blue   = 7;
}
```

В директории [include/ssl-client/protobuf-files/pb/proto/](https://github.com/robocin/ssl-client/tree/master/include/ssl-client/protobuf-files/pb/proto) можно найти аналогичные списки параметров для всех стандартных пакетов. Тоже самое находится на [официальном сайте лиги](https://ssl.robocup.org/league-software/#:~:text=Standard%20Network%20Parameters-,Protobuf%20Definitions,-There%20are%20several).

## Управление роботом в симуляторе
Тут же ([строки 86-90](https://github.com/robocin/ssl-client/blob/master/src/main.cpp#L86-L90)) посылаются команды на синих роботов, чтобы показать, что зрение работает:

``` C++
if (robot.x() <= 0) {
    grSim_client.sendCommand(1.0, i);
} else {
    grSim_client.sendCommand(-1.0, i);
}
```

*sendCommand(velX, id)* - простая функция, которая отправляет на робота с индексом *id*, скорость движения вперед:

``` C++
void GrSim_Client_Example::sendCommand(double velX, int id) {
    double zero = 0.0;
    grSim_Packet packet;  //пакет, который будет передан grSim через протокол Google Protobuf
    bool yellow = false;
    packet.mutable_commands()->set_isteamyellow(yellow);
    packet.mutable_commands()->set_timestamp(0.0);
    grSim_Robot_Command* command =
        packet.mutable_commands()->add_robot_commands();  //список команд на роботов
    command->set_id(id);

    command->set_wheelsspeed(!true);  //возможность управления колесами по отдельности
    command->set_wheel1(zero);
    command->set_wheel2(zero);
    command->set_wheel3(zero);
    command->set_wheel4(zero);
    command->set_veltangent(velX);  //скорость движения
    command->set_velnormal(zero);
    command->set_velangular(zero);

    command->set_kickspeedx(zero);  //удар прямо
    command->set_kickspeedz(zero);  //удар навесом
    command->set_spinner(false);  //дрибблер

    QByteArray dgram;
    dgram.resize(packet.ByteSize());
    packet.SerializeToArray(dgram.data(), dgram.size());  //создание массива для отправки
    if (socket->writeDatagram(dgram, this->_addr, this->_port) > -1) {  //отправка
        qDebug("send data");
    }
}
```

Видно, что функция забивает нулями все неиспользуемые параметры, передаваемые на робота.

## Получение команд от [Game Controller](../software/programs.md#ssl-game-controller)

Получение команд от судей не показано в файле <code>main.cpp</code>, но реализовано в файле [.../referee_ssl_client.cpp](https://github.com/robocin/ssl-client/blob/master/include/ssl-client/net/referee_ssl_client/referee_ssl_client.cpp). Работает аналогично с получением пакетов от SSL Vision

Для того чтобы подробнее узнать о коммуникации внутри сети в лиге RoboCup SSL, изучите исходный код [описанного репозитория](https://www.github.com/robocin/ssl-client). Если остались вопросы, не стесняйтесь связываться [любым удобным способом](../links.md) с нами и спрашивать лично, будем рады помочь!

Спасибо за проявленный интерес к лиге!
