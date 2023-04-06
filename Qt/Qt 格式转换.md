记录下Qt中常见数据类型、结构的相互转换

## 1. QString 与 String 的转换
```
//QString 转 String
QString qstr = "hello";
string str = qstr.toStdString();

//String 转 QString
string str = "hello";
QString qstr = QString::fromStdString(str);
```

## 2. QString 与 int 的转换
```
//QString 转 int
QString str = "100";
int tmp = str.toInt();

//int 转 QString
int tmp = 100;
QString str = QString::number(tmp);
```

## 3. QString 与 QByteArray 的转换
```
//QString 转 QByteArray
QString str = "hello";
QByteArray byte = str.toUtf8();

QString str = "hello";
QByteArray byte = str.toStdString().c_str();

//QByteArray 转 QString
QByteArray byte("hello");
QString str = byte;

QByteArray byte("hello");
QString str;
str.prepend(byte);
```

## 4. QImage 与 QByteArray 的转换
```
//QImage 转 QByteArray
QImage image;
QByteArray byte;
QBuffer buffer(&byte);
buffer.open(QIODevice::Append);
image.save(&buffer, "JPG");

//QByteArray 转 QImage
QByteArray byte;       
QBuffer buffer(&byte);       
buffer.open(QIODevice::ReadOnly);       
QImageReader reader(&buffer,"JPG");       
QImage image = reader.read();  

//使用QImage的构造函数，有多个重载，具体可以参考Qt文档
QImage image = QImage((unsigned char*)byte.data(), width, heigth, bytesPerLine, QImage::Format);
```

## 5. QByteArray 与 自定义结构体 的转换
自定义结构体名字为MY_STRUCT
```
//QByteArray 转 结构体
QByteArray byte;
MY_STRUCT *p = reinterpret_cast<MY_STRUCT *>(byte.data());

//结构体 转 QByteArray
QByteArray byte;
byte.append((char*)&p, sizeof(MY_STRUCT));
```

## 6. QByteArray 与 char * 的转换
```
//QByteArray 转 char *
char *ch;
QByteArray byte;
ch = byte.data();

//char * 转 QByteArray
char *ch;
QByteArray byte;
byte = QByteArray(ch);
```

## 7. QString 与 char * 的转换
```
//QString 转 char *
QString str;
char *ch;
QByteArray byte = str.toUtf8();
ch = byte.data();

QString str;
std::string string = str.toStdString();
const char *ch = string.c_str();

//char * 转 QString
const char *ch = "hello";
QString str(ch);

const char *ch = "hello";
QString str = QString::fromUtf8(ch);
```