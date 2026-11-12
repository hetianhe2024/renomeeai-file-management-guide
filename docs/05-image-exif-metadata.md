# 用 EXIF 元数据重命名照片

Rename photos by EXIF data 依赖的是拍摄时写入文件的元数据，不是资源管理器里的“创建时间”。把照片从存储卡复制到电脑、用聊天软件转发、或另存为网页用的 JPEG，文件系统时间都会变。EXIF 里的拍摄时间在文件字节没被改写时保持原值。批量归档应该读这个字段。

EXIF（Exchangeable Image File Format）是图像文件中的元数据标准，相机、手机和多数 RAW 转换器都会写入。JPEG 把它放在 APP1 段；HEIC 和许多 RAW 容器也带有同一套字段。截图、部分即时通讯软件导出的图片，以及“存储为 Web 所用格式”，经常没有这块数据。

## 命名时真正有用的字段

| 字段 | 十进制标签 | 含义 | 是否适合直接进入文件名 |
| --- | --- | --- | --- |
| `DateTimeOriginal` | 36867 | 拍摄时间，形如 `2024:03:15 14:30:22` | 适合，作为默认时间 |
| `DateTimeDigitized` | 36868 | 数字化时间 | 扫描老照片时有用 |
| `DateTime` | 306 | 文件最近一次被修改的时间 | 不适合代替拍摄时间 |
| `Make` / `Model` | 271 / 272 | 厂商与机身型号 | 多机位拍摄时适合 |
| `LensModel` | 42036 | 镜头型号 | 适合技术归档，名字会变长 |
| `GPSLatitude` 与 `GPSLongitude` | GPS IFD | 纬度、经度，度分秒有理数 | 适合坐标；它不是城市名 |
| `Orientation` | 274 | 显示方向 | 只影响显示，不要写进文件名 |

标签号见 ExifTool 的 EXIF 标签表[^exifTags]。

[^exifTags]: <https://exiftool.org/TagNames/EXIF.html>

`DateTime` 会在编辑软件保存时更新。按拍摄顺序重命名时只使用 `DateTimeOriginal`。三个时间字段不一致时，以 `DateTimeOriginal` 为准，并把另外两个留在目录里备查，不要取平均值。

GPS 存的是坐标和南纬/北纬、东经/西经标记。文件名里的 `shanghai` 来自反向地理编码或人工标注，不是 EXIF 原字段。反向地理编码需要一份地名数据或在线服务，坐标相同的两张照片才会得到同一个地名。没有做这一步时，应把坐标省略，或保留四位左右的小数，而不是猜一个城市[^gpsTags]。

[^gpsTags]: <https://exiftool.org/TagNames/GPS.html>

## 读取

ExifTool 可以一次打印命名需要的字段[^exiftool]：

[^exiftool]: <https://exiftool.org/>

```text
exiftool -DateTimeOriginal -Model -GPSLatitude -GPSLongitude photo.jpg
```

批量导出成表格后再检查空值：

```text
exiftool -csv -DateTimeOriginal -Model -FileName -ext jpg -ext heic . > exif.csv
```

Windows 资源管理器的“拍摄日期”列对应的就是这类拍摄时间，可以用来核对，不适合作为唯一的批量改名工具：它不能按“日期 + 机身 + 序号”组成模板。macOS 的“显示简介”能看单张照片的 EXIF，同样不适合几百张的模板命名。

空值要单独处理。`DateTimeOriginal` 为空的文件，多半是截图、转发图或被剥掉元数据的副本。给它们加上可筛选的标记，例如保留原名并在清单里记 `NO-EXIF`，比悄悄改用“今天的修改时间”更不容易污染时间线。

## 组成文件名

把 EXIF 的日期时间改成可排序的形式：

```text
2024:03:15 14:30:22   →   2024-03-15_143022
```

单机位、只关心时间：

```text
2024-03-15_143022.jpg
```

同一秒连拍会冲突。按原始文件名排序后追加序号：

```text
2024-03-15_143022_01.jpg
2024-03-15_143022_02.jpg
```

多机位再加机身，空格换成连字符，去掉商标里的特殊字符：

```text
2024-03-15_143022_canon-eos-r5.jpg
```

重命名只改变目录项的名称。图像像素、色彩和 EXIF 字节都应保持原样。可靠的检查方法是改名前后各算一次内容哈希，或用 ExifTool 再读一次 `DateTimeOriginal`，两次应相同。会“顺手校正方向并把 Orientation 烘焙进像素”的导出功能，不属于重命名。

分享出去的副本要另外考虑 GPS。文件名如果含经纬度，等于把拍摄地点写成了明文；原片里的 GPS IFD 仍然在。对外发送时使用去掉 GPS 的副本，归档原片保留坐标。这是两份文件，不是同一次重命名的两个选项。

## 哪些文件读不到 EXIF

- 系统截图、扫描件导出的 JPEG、没有嵌入 EXIF 的 PNG。
- 社交媒体和部分聊天工具重新压缩后的图片。它们生成的是新文件。
- 只保留了侧车文件（sidecar）的 RAW 工作流。元数据可能在 `.xmp` 里，而不在图像文件里。重命名时要把侧车文件一起改成同一个主名，否则返回软件会找不到设置。
- HEIC。字段通常在，但只支持 JPEG 的旧脚本会跳过它们。脚本应明确列出 `jpg`、`jpeg`、`heic`、`tif` 以及你实际使用的 RAW 扩展名。

## 在 RenomeeAI 里怎么用这些字段

拍摄日期、机身和 GPS 坐标都属于元数据重命名，不需要 OCR，也不需要模型描述画面里“这是一场婚礼”。官方 FAQ 写明：按拍摄日期读取 EXIF 可以离线完成，且不上传文件内容；需要模型识别画面或图片中的文字时才联网，并要在应用内确认[^renomeeFaq]。功能页的字段表包含拍摄时间、相机、GPS 和分辨率，格式包含 JPG、HEIC、WEBP、TIFF[^renomeeFeatures]。只改文件名、不改图像数据，是这一功能的边界。

[^renomeeFaq]: <https://renomeeai.readthedocs.io/en/latest/en/faq/>
[^renomeeFeatures]: <https://renomeeai.com/en/product/features/>

按字段生成文件名的说明见官网[^renamePhotosByExif]。预览里如果出现城市名，而你的指令只要求日期和机身，应把地名从规则里去掉，回到 EXIF 原字段。

[^renamePhotosByExif]: <https://renomeeai.com/en/rename-files/photo/rename-photos-by-exif-data/>

画面内容、水印文字和票据金额属于 OCR 或视觉识别，不是 EXIF。那些文件留到 OCR 一章，不要为了补一个空的 `DateTimeOriginal` 去调用识别。
