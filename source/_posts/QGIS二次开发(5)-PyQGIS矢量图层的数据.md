---
title: QGIS二次开发(5)-PyQGIS矢量图层的数据
date: 2025-11-13 20:30:19
tags:
  - GIS
  - QT
categories:
  - QGIS二次开发
---

>本节主要介绍矢量图层的数据组成，包括属性、几何体和数据源三个层面，一起解构QGIS中对矢量数据的抽象吧！

# 要素的属性

要素是构成矢量数据模型的基本单元。 点图层中的一个点，线图层中的一条线，多边形图层中的一个面，都是以要素为单元进行描述。

属性是要素的重要组成部分，主要用于描述要素的非空间特征，另一个重要组成则是几何体。可以将图层想象为一个二维的表。每一列代表一个属性，每个属性都有各自的类型，几何体也可以理解为一个独立的属性。 每一行则是一个具体的要素。基于数据表的这种理解方 式，很多时候我们可以用`SQL`中的**字段**来等价的替代**属性**。

## 属性类型

QGIS中的要素属性类型有如下几种：

|     类型      |       用途       |        示例        |
| :-----------: | :--------------: | :----------------: |
| **文本数据**  |    存储字符串    |     名称、地址     |
|   **整数**    |     存储整数     |   ID、人口、计数   |
| **小数/实数** | 存储带小数的数字 |  面积、长度、坐标  |
| **日期/时间** |   存储时间信息   | 创建日期、调查时间 |
|  **布尔值**   |   存储是/否值    | 是否可见、是否有效 |
|   **BLOB**    |     存储文件     |     照片、文档     |

根据数据源格式的不同，某些属性类型会不被支持或者会有新的属性类型出现。如BLOB类型，其底层使用二进制的形式存储数据，geojson格式在存储层面对此属性类型的支持不够完善。

- 文本数据： 用于存储文本格式的数据，对应字符串
- 整数： 用于存储正负整数
- 实数： 用于存储实数，对应于 浮点数
- 在定义属性是，应根据需要选择合适的类型，存放合适的数据

## QgsField

```c++
    /**
     * Constructor. Constructs a new QgsField object.
     * \param name Field name
     * \param type Field variant type, currently supported: String / Int / Double
     * \param typeName Field type (e.g., char, varchar, text, int, serial, double).
     * Field types are usually unique to the source and are stored exactly
     * as returned from the data store.
     * \param len Field length
     * \param prec Field precision. Usually decimal places but may also be
     * used in conjunction with other fields types (e.g., variable character fields)
     * \param comment Comment for the field
     * \param subType If the field is a collection, its element's type. When
     *                all the elements don't need to have the same type, leave
     *                this to QVariant::Invalid.
     */
    QgsField( const QString &name = QString(),
              QVariant::Type type = QVariant::Invalid,
              const QString &typeName = QString(),
              int len = 0,
              int prec = 0,
              const QString &comment = QString(),
              QVariant::Type subType = QVariant::Invalid );
```

|     参数名     | 是否必需 |                             描述                             |                  示例                   |
| :------------: | :------: | :----------------------------------------------------------: | :-------------------------------------: |
|   **`name`**   |  **是**  |                    字段的名称（字符串）。                    |        `"population"`, `"名称"`         |
|   **`type`**   |  **是**  |            字段的数据类型，使用 `QVariant` 枚举。            |    `QVariant.String`, `QVariant.Int`    |
| **`typeName`** |    否    | 数据类型的字符串名称。通常留空，系统会根据 `type` 自动设置。 |          `"text"`, `"integer"`          |
|   **`len`**    |    否    |        字段长度（整数）。对文本字段和整数字段很重要。        | `255`（文本最大长度），`10`（整数位数） |
|   **`prec`**   |    否    |     字段精度（整数）。对于小数类型，表示小数点后的位数。     |         `2`（表示保留两位小数）         |
| **`comment`**  |    否    |                 字段的注释或别名（字符串）。                 |           `"2020年常住人口"`            |
| **`subType`**  |    否    | 更细粒度的子类型，如 `QgsField.NoneSubType`， `QgsField.Integer64`等。 |           `QVariant.LongLong`           |

### type 和 typeName

大家需要注意到，type和typeName是两个不同的参数。 

- 事实上，type是类型的抽象，具有跨平台的特性，不依赖于具体的文件格式或者数据库类型。如整数，type没有具体的描述是INT8 INT32 INT64等
- typeName是一个字符串，代表类型的具体的实现，根据底层数据源的不同，会需要有不同的映射。 
  - **Shapefile**： 使用 `"Integer"`， `"Real"`， `"String"`， `"Date"`。
  - **GeoPackage/SQLite**： 使用 `"INTEGER"`， `"REAL"`， `"TEXT"`， `"BLOB"`。
  - **PostGIS**： 使用 `"int4"`， `"varchar"`， `"float8"`， `"timestamp"`。

- 一般来说，typeName不需要填写。但需要针对特定的数据源做底层定制时，可以使用该参数

  ```python
  # 明确指定 type 和 typeName
  field = QgsField("name", QVariant.String, "varchar", 255)
  ```

### 使用QgsField

```python
fields = QgsFields()
fields.append(QgsField("ID", QVariant.Int, len=10)) # 整数字段，长度10
fields.append(QgsField("名称", QVariant.String, len=50)) # 文本字段，最大长度50
fields.append(QgsField("人口", QVariant.Int, len=10))
fields.append(QgsField("面积", QVariant.Double, len=10, prec=2)) # 小数字段，总长10，小数点后2位

feat = QgsFeature()
feat.setFields(fields) # 让要素知道字段结构
feat.setAttributes([1, "北京", 21540000, 16410.54]) # 设置属性值，顺序与字段定义一致

feat.setAttribute("ID",2)
print(feat.attribute("ID"))
## 2
```

- 定义一个QgsField集合
- 按顺序添加多个属性
- 将属性设置给要素
- 设置、修改和获取要素属性对应的值
- 至此我们可以更直观的理解要素属性，要素属性是存放在要素本身的键值对(map,dict 等不同语言有不同的描述)，属性名称是key，属性类型则规定了value的类型

## 图层属性与要素属性

QGIS中矢量图层和要素都可以定义属性类型，他们之前存在默认约束行为。

- 当要素独立存在时，可以将要素作为一个键值对使用，自定义其要素类型并存入和取出其对应的值
- 当要素依附于矢量图层时，必须符合矢量图层的属性定义。也就是说，矢量图层定义的属性类型是一种结构性约束，是一种蓝图，要求存放在该图层中的要素必须具有同样的属性。
- 从属性类型约束的角度来理解另外一个事实，几何体类型。当我们把几何体类型也作为属性类型看待时就可以很好的理解为什么图层需要区分点、线、面等集合类型。

# 几何体

