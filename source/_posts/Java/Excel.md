---
title: 使用POI解析Excel
date: 2018-09-08
categories: JAVA
---

Excel作为一种常用的数据存储格式，在很多项目中都会有相应的导入导出的功能。这篇文章会介绍如何使用Java操作Excel，以及如何解决大文件读写时内存溢出的问题。


##1、OpenXML标准

[Word](https://en.wikipedia.org/wiki/Microsoft_Word)、[Excel](https://en.wikipedia.org/wiki/Microsoft_Excel)、[PPT](https://en.wikipedia.org/wiki/Microsoft_PowerPoint)是Office办公套件中最常用的三个组件。早期的Office套件使用二进制格式，这里面包括以[`.doc`](https://en.wikipedia.org/wiki/Doc_%28computing%29)、[`.xls`](https://en.wikipedia.org/wiki/Microsoft_Excel_file_format)、[`.ppt`](https://en.wikipedia.org/wiki/Microsoft_PowerPoint#File_formats)为后缀的文件；直到07这个划时代的版本将基于XML的压缩格式作为默认文件格式，也就是相应以`.docx`、`.xlsx`、`.pptx`为后缀的文件。

这个结合了XML与Zip压缩技术的新文件格式使用的是[OpenXML](https://en.wikipedia.org/wiki/Office_Open_XML)标准。微软从2000年开始酝酿这项技术标准，到2006年申请成为[ECMA-376](http://www.ecma-international.org/publications/standards/Ecma-376.htm)，然后在Office2007中用作默认的文件格式，再到08年成为了[ISO / IEC 29500](https://www.iso.org/standard/51463.html)国际标准，后续每两三年就会发布一个新版本。Office的一路凯歌无不彰显微软雄厚的实力。

> 所以说三流公司做产品，二流公司做平台，一流公司定标准。

[微软的官方文档中](https://docs.microsoft.com/en-us/office/open-xml/open-xml-sdk)详细介绍了[WordprocessingML(Word)](https://docs.microsoft.com/en-us/office/open-xml/working-with-wordprocessingml-documents)、[SpreadsheetML(Excel)](https://docs.microsoft.com/en-us/office/open-xml/working-with-spreadsheetml-documents)、[PresentationML(PPT)](https://docs.microsoft.com/en-us/office/open-xml/working-with-presentationml-documents)三个标准，这里主要介绍Excel的部分内容。

首先Excel几个最基础的概念：

* 一个Excel就是一个工作簿(Workbook)
* 一个Sheet就是一张表格
* 一个Workbook可以包含多个Sheet
* 每一行Row的每一列就是一个单元格(Cell)

![Exce基础结构](http://tva1.sinaimg.cn/large/bda5cd74gy1fv1hzbd7zmj20hh09fdga.jpg)

因为07版后的`.xlsx`本质上就是一个压缩包，我们完全可以用解压工具打开它。

一个基础的Excel解压之后，目录结构大致如下：

![Excel文件解压后的结构](http://tva1.sinaimg.cn/large/bda5cd74gy1fv1hhpzainj20z40kjdkd.jpg)

更典型的Excel还包括：数字、文本、公式、图表(Chart)、普通列表(Table)、数据透视表(Pivot Table)等内容。

![Excel](http://tva1.sinaimg.cn/large/bda5cd74gy1fv2gw6q03yg20kd0lntag.gif)

> Excel远比我们想象的复杂

##2、使用POI操作Excel

Java领域最常见的两个操作Excel的工具库分别是[JXL(Java Excel API)](http://jexcelapi.sourceforge.net/)和Apache的[POI](http://poi.apache.org/)。JXL有个严重的缺点就是只支持07版本之前的二进制格式Excel，而POI除了能操作Excel，还可以操作Word和PPT以及Office套装中其他的组件，高下立现。

> POI全称是Poor Obfuscation Implementation，简洁模糊实现，也有人翻译成糟糕的模糊实现。

POI目前最新版本是4.0，可以将相应maven依赖添加到pom.xml文件中：

```xml
<!-- poi:07版之前的二进制格式 -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi</artifactId>
    <version>4.0.0</version>
</dependency>
<!-- poi-ooxml:07版之后的OpenXML格式 -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>4.0.0</version>
</dependency>
```

![ComponentMap](http://tva1.sinaimg.cn/large/bda5cd74gy1fv20whtg4pj20ut0alt9c.jpg)

[POI提供了三种读写Excel的方式](http://poi.apache.org/components/spreadsheet/)：

![HSSF,XSSF,SXSSF](http://tva1.sinaimg.cn/large/bda5cd74gy1fv1j62m9tfj20g2079t8s.jpg)

1、HSSF支持`.xls`为后缀的二进制格式，并提供了流解析模式的`HSSFListener`相关API以及基于内存模型的`HSSFWorkbook`相关API。

2、XSSF支持`.xlsx`为后缀的OpenXML格式。因为是底层文件是XML所以可以使用SAX解析，POI提供了`XSSFReader`用来获取压缩包中的各个XML文件相应的输入流；另外提供了基于DOM解析模式的[`XSSFWorkbook`](https://poi.apache.org/apidocs/org/apache/poi/xssf/usermodel/XSSFWorkbook.html)相关API。

3、POI3.8后提供了[SXSSF API](https://poi.apache.org/apidocs/org/apache/poi/xssf/streaming/SXSSFWorkbook.html)，它是基于XSSF构建的低内存占用版本(使用滑动窗口机制来实现低内存访问)。但是需要注意的是**SXSSFWorkbook默认使用内联字符串而不是[共享字符串表(SharedStringsTable)](https://docs.microsoft.com/zh-cn/office/open-xml/working-with-the-shared-string-table)**，这样可以让保存在内存中的数据尽可能更少(SharedStringsTable需要常驻内存)，所以如果是自己写SAX解析要注意兼容性。

> POI滑动窗口只窗口范围内的单元格数据加载到内存中，窗口外的数据读写内容会以临时文件的形式保存到磁盘上，同时还支持临时文件的压缩。SXSSF可以通过构造函数中的`rowAccessWindowSize`参数指定窗口大小，`compressTmpFiles`指定是否压缩临时文件，`useSharedStringsTable`指定是否使用共享字符表。

## 2.1、使用Workbook API

上面说的三种方式都有一个Workbook实现类，用法上基本一致。唯一不同的是SXSSFWorkbook最后需要调用`dispose()`方法处理磁盘上的临时文件。

![Workbook继承图](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuShCAqajIajCJbK8po_AJihFp-Q2CHHiQdHr5Jo2WzrmI4NWbWf6aND8pKi1MWO0)

下面是使用XSSFWorkbook读取`.xlsx`文件的例子：

```java
// 打开指定位置的Excel文件
FileInputStream file = new FileInputStream(new File(fileLocation));
Workbook workbook = new XSSFWorkbook(file);
// 打开Excel中的第一个Sheet
Sheet sheet = workbook.getSheetAt(0);

// 读取Sheet中的数据
Map<Integer, List<String>> data = new HashMap<>();
int i = 0;
for (Row row : sheet) { // 行
    data.put(i, new ArrayList<String>());
    for (Cell cell : row) { // 单元格
        switch (cell.getCellType()) { // 不同的数据类型
            case STRING: ... break; // 字符串类型
            case NUMERIC: ... break; // 数值类型
            case BOOLEAN: ... break; // 布尔类型
            case FORMULA: ... break; // 公式类型
            case BLANK: ... break; // 空白类型
        }
    }
    i++;
}
```

POI有不同的方法来读取每种类型的数据：

```java
switch(cell.getCellType()) {
    case CellType.STRING：
        data.get(i).add(cell.getRichStringCellValue().getString());
        break;
    case CellType.NUMERIC：
        if(DateUtil.isCellDateFormatted(cell)) {
            data.get(i).add(cell.getDateCellValue));
        } else {
            data.get(i).add(cell.getNumericCellValue());
        }
        break;
    case CellType.BOOLEAN：
        data.get(i).add(cell.getBooleanCellValue());
        break;
    case CellType.FORMULA：
        data.get(i).add(cell.getCellFormula());
        break;
    case CellType.BLANK：
        data.get(i).add("")
        break;
}
```

Workbook API也支持Excel的写入：

```java
Workbook workbook = new XSSFWorkbook(); // 创建工作簿

Sheet sheet = workbook.createSheet("Persons"); // 创建Sheet
sheet.setColumnWidth(1, 4000);

Row header = sheet.createRow(0); // 创建表头行

CellStyle headerStyle = workbook.createCellStyle(); // 表头单元格样式
headerStyle.setFillForegroundColor(IndexedColors.LIGHT_BLUE.getIndex());
headerStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);
XSSFFont font = ((XSSFWorkbook) workbook).createFont(); // 字体样式
font.setFontName("Arial");
font.setFontHeightInPoints((short) 16);
font.setBold(true);
headerStyle.setFont(font);

Cell headerCell = header.createCell(0); // 创建表头单元格
headerCell.setCellValue("Name");
headerCell.setCellStyle(headerStyle);

headerCell = header.createCell(1); // 创建表头单元格
headerCell.setCellValue("Age");
headerCell.setCellStyle(headerStyle);

CellStyle style = workbook.createCellStyle(); // 普通单元格样式
style.setWrapText(true);

Row row = sheet.createRow(2); // 写入单元格
Cell cell = row.createCell(0);
cell.setCellValue("John Smith");
cell.setCellStyle(style);

cell = row.createCell(1); // 写入单元格
cell.setCellValue(20);
cell.setCellStyle(style);

// 最后写出到文件
FileOutputStream outputStream = Files.newOutputStream("/path/to/excel");
workbook.write(outputStream);
workbook.close();
```

POI还支持插入图片、形状、数据透视表以及更多的样式，更多详细代码可以参考[官方的快速入门指南](http://poi.apache.org/components/spreadsheet/quick-guide.html)

## 2.2、HSSFListener实现流式解析

虽然SXSSFWorkbook通过滑动窗口有效地降低了内存消耗，但是并不支持读的功能，而且写功能也只支持OpenXML格式。而HSSFWorkbook和XSSFWorkbook需要将Excel内容全部读取到内存才能操作，对于二进制Excel大文件的读取必须使用HSSFListener。

> 不过03版二进制Excel能支持的最大行数为65536

```java
// 使用EventAPI读取Excel
public class EventExample implements HSSFListener
{
    private SSTRecord sstrec;

    // 实现接口方法
    public void processRecord(Record record)
    {
        switch (record.getSid()) {
            case BOFRecord.sid: // Beginning Of File
                BOFRecord bof = (BOFRecord) record;
                if (bof.getType() == bof.TYPE_WORKBOOK) {
                    System.out.println("Encountered workbook");
                } else if (bof.getType() == bof.TYPE_WORKSHEET) {
                    System.out.println("Encountered sheet reference");
                }
                break;
            case BoundSheetRecord.sid:
                BoundSheetRecord bsr = (BoundSheetRecord) record;
                System.out.println("New sheet named:" + bsr.getSheetname());
                break;
            case RowRecord.sid: // 行
                RowRecord rowrec = (RowRecord) record;
                System.out.println("first column:" + rowrec.getFirstCol() + ","
                                   "last column:" + rowrec.getLastCol());
                break;
            case NumberRecord.sid: // 数字单元格
                NumberRecord numrec = (NumberRecord) record;
                System.out.println("Row:"+numrec.getRow() + ","
                                   "Column:" + numrec.getColumn() + ","
                                   "Number value:" + numrec.getValue());
                break;
            case SSTRecord.sid: // Static String Table Record
                sstrec = (SSTRecord) record;
                System.out.println("String table value:");
                for (int k = 0; k < sstrec.getNumUniqueStrings(); k++) {
                    System.out.println(k + " = " + sstrec.getString(k));
                }
                break;
            case LabelSSTRecord.sid:
                LabelSSTRecord lrec = (LabelSSTRecord) record;
                System.out.println("String cell value:" + sstrec.getString(lrec.getSSTIndex()));
                break;
        }
    }

    public static void main(String[] args) throws IOException
    {
        FileInputStream fin = new FileInputStream("/path/to/file");
        POIFSFileSystem poifs = new POIFSFileSystem(fin);
        InputStream din = poifs.createDocumentInputStream("Workbook");
        // 构造 HSSFRequest 对象
        HSSFRequest req = new HSSFRequest();
        // 监听所有的Record
        req.addListenerForAllRecords(new EventExample());
        // 创建EventFactory
        HSSFEventFactory factory = new HSSFEventFactory();
        // 将输入流交给EventFactory解析生成事件
        factory.processEvents(req, din);
        // 事件处理完后关闭输入流
        fin.close();
        din.close();
        System.out.println("done.");
    }
}
```

![Record继承图](http://tva1.sinaimg.cn/large/bda5cd74gy1fv2c87sd90j21b90b4q3s.jpg)

## 2.3、SXSSF API

SXSSF(org.apache.poi.xssf.streaming)是兼容XSSF API的流式扩展，用于生成数据量较大的Excel文件。SXSSF通过限制滑动窗口内行的访问来实现低内存占用，而XSSF API允许访问文档中的所有行。不在窗口中的旧行将不可访问，因为它们已经被写入磁盘。

您可以通过`new SXSSFWorkbook(int windowSize)`指定窗口大小， 也可以通过`SXSSFSheet.setRandomAccessWindowSize(int windowSize)`设置每个sheet的窗口大小

当通过`createRow()`创建新行时，且尚未`flush()`的数据总量超过指定的窗口大小时，将`flush()`内存中最前面的行，并且不能再通过`getRow()`访问该行。

**默认窗口大小为100**，由`SXSSFWorkbook.DEFAULT_WINDOW_SIZE`定义。

**windowSize为-1表示无限制访问**。在这种情况下，所有未调用`flushRows()`强制刷新的记录都可访问。

请注意，SXSSF会创建临时文件，所以**最后必须通过调用`dispose()`方法来显式清理临时文件**。

**SXSSFWorkbook默认使用内联字符串而不是共享字符串表**。因为不需要在内存中保存文档字符内容，所以这种方式能有效地减低内存消耗，但是**也导致了生成与某些客户端不兼容的文档**。启用共享字符串后，文档中所有的唯一字符串都必须保留在内存中，所以这可能会比禁用共享字符串消耗更多的资源。

另外还需要注意，根据你使用的功能，仍然可能消耗大量内存，例如合并区域，超链接，注释......，这些内容只存储在内存中。

在决定是否启用共享字符串之前，请仔细检查内存预算和兼容性需求。

SXSSF在将sheet的数据刷新到临时文件中(每个sheet一个临时文件)，这些临时文件可能会变得非常大。比如，对于20 MB的csv数据，临时xml文件将超过GB字节。如果临时文件的大小是个问题，可以通过`setCompressTempFiles(true)`让SXSSF使用gzip压缩。

```java
// 只在内存中保存100, 超出的行将会flush到磁盘
SXSSFWorkbook wb = new SXSSFWorkbook(100);
Sheet sh = wb.createSheet();
// 写入1000行数据
for(int rownum = 0; rownum < 1000; rownum++){
    Row row = sh.createRow(rownum);
    for(int cellnum = 0; cellnum < 10; cellnum++){
        Cell cell = row.createCell(cellnum);
        String address = new CellReference(cell).formatAsString();
        cell.setCellValue(address);
    }

}

// 前900行已经flush到磁盘了，无法访问
for(int rownum = 0; rownum < 900; rownum++){
    Assert.assertNull(sh.getRow(rownum));
}

// 最后的100行仍可以访问
for(int rownum = 900; rownum < 1000; rownum++){
    Assert.assertNotNull(sh.getRow(rownum));
}

FileOutputStream out = new FileOutputStream("/temp/sxssf.xlsx");
wb.write(out);
out.close();

// 处理磁盘上的临时文件
wb.dispose();
```

我们还可以将windowSize置为-1，通过调用`SXSSFSheet.flushRows()`手动刷新到磁盘

```java
// 置为-1，关闭自动flush
SXSSFWorkbook wb = new SXSSFWorkbook(-1);
Sheet sh = wb.createSheet();

for(int rownum = 0; rownum < 1000; rownum++){
    Row row = sh.createRow(rownum);
    for(int cellnum = 0; cellnum < 10; cellnum++){
        Cell cell = row.createCell(cellnum);
        String address = new CellReference(cell).formatAsString();
        cell.setCellValue(address);
    }

    // 手动控制flush
    if(rownum % 100 == 0) {
        // 保留最后100行数据，其余全部flush到磁盘
        ((SXSSFSheet)sh).flushRows(100);
        // ((SXSSFSheet)sh).flushRows() == ((SXSSFSheet)sh).flushRows(0),
    }

}

FileOutputStream out = new FileOutputStream("/temp/sxssf.xlsx");
wb.write(out);
out.close();

// 处理磁盘上的临时文件
wb.dispose();
```

## 2.4、使用SAX解析xlsx文件

虽然大文件的写入有SXSSF的支持，但是读取暂时没有更好的解决方案。POI目前推荐的做法是直接使用SAX API手动解析XML。这要求开发者对Excel的接口有清楚的认识。

![Sheet与SharedStrings文件结构](http://tva1.sinaimg.cn/large/bda5cd74gy1fv28s5zo2yj211w0idtaa.jpg)

POI也对SAX解析提供了一些支持——[XSSFReader](https://poi.apache.org/apidocs/org/apache/poi/xssf/eventusermodel/XSSFReader.html)。

XSSFReader能帮我们轻松地获取`.xlsx`压缩包中各个部分的输入流，

> XSSFReader有一个子类[XSSFBReader](https://poi.apache.org/apidocs/org/apache/poi/xssf/eventusermodel/XSSFBReader.html)用于读取[`.xlsb`文件](https://www.spreadsheet1.com/how-to-save-as-binary-excel-workbook.html)。

```java
public class ExampleEventUserModel {
	public void processOneSheet(String filename) throws Exception {
		OPCPackage pkg = OPCPackage.open(filename);
		XSSFReader r = new XSSFReader(pkg);
		SharedStringsTable sst = r.getSharedStringsTable();

		XMLReader parser = fetchSheetParser(sst);

		// 根据 Sheet Name / Sheet Order / rID 查找相应的Sheet
		// 你只需要使用SAX API处理相应的输入流
		// 通常情况使用 rId# 或 rSheet# 形式的id
		InputStream sheet2 = r.getSheet("rId2");
		InputSource sheetSource = new InputSource(sheet2);
		parser.parse(sheetSource);
		sheet2.close();
	}

	public void processAllSheets(String filename) throws Exception {
		OPCPackage pkg = OPCPackage.open(filename);
		XSSFReader r = new XSSFReader(pkg);
		SharedStringsTable sst = r.getSharedStringsTable();

		XMLReader parser = fetchSheetParser(sst);

		Iterator<InputStream> sheets = r.getSheetsData();
		while(sheets.hasNext()) {
			System.out.println("Processing new sheet:\n");
			InputStream sheet = sheets.next();
			InputSource sheetSource = new InputSource(sheet);
			parser.parse(sheetSource);
			sheet.close();
			System.out.println("");
		}
	}

	public XMLReader fetchSheetParser(SharedStringsTable sst) throws SAXException {
		XMLReader parser =
			XMLReaderFactory.createXMLReader(
					"org.apache.xerces.parsers.SAXParser"
			);
		ContentHandler handler = new SheetHandler(sst);
		parser.setContentHandler(handler);
		return parser;
	}
    // 使用SAX API解析XML
	private static class SheetHandler extends DefaultHandler {
		private SharedStringsTable sst; // 共享字符串表
		private String lastContents;
		private boolean nextIsString;

		private SheetHandler(SharedStringsTable sst) {
			this.sst = sst;
		}

		public void startElement(String uri, String localName, String name,
				Attributes attributes) throws SAXException {
			// c => cell
			if(name.equals("c")) {
				// r => reference
				System.out.print(attributes.getValue("r") + " - ");
				// t => type
				String cellType = attributes.getValue("t");
                // s => string
				if(cellType != null && cellType.equals("s")) {
					nextIsString = true;
				} else {
					nextIsString = false;
				}
			}
			// 清空上一次的内容
			lastContents = "";
		}

		public void endElement(String uri, String localName, String name)
				throws SAXException {
			if(nextIsString) {
                // sheet中存储了共享字符表的索引
				int idx = Integer.parseInt(lastContents);
				lastContents = new XSSFRichTextString(sst.getEntryAt(idx)).toString();
				nextIsString = false;
			}

			// v => contents of a cell
			if(name.equals("v")) {
				System.out.println(lastContents);
			}
		}

		public void characters(char[] ch, int start, int length)
				throws SAXException {
			lastContents += new String(ch, start, length);
		}
	}

	public static void main(String[] args) throws Exception {
		ExampleEventUserModel example = new ExampleEventUserModel();
		example.processOneSheet(args[0]);
		example.processAllSheets(args[0]);
	}
}
```

上面的代码是使用原生SAX API进行XML处理的例子，它要求我们知道sheet.xml文件的内容结构。POI已经将这部分逻辑封装在了[`XSSFSheetXMLHandler`](https://poi.apache.org/apidocs/org/apache/poi/xssf/eventusermodel/XSSFSheetXMLHandler.html)中，我们只要实现它暴露的[`SheetContentsHandler`](https://poi.apache.org/apidocs/org/apache/poi/xssf/eventusermodel/XSSFSheetXMLHandler.SheetContentsHandler.html)接口即可。

使用SheetContentsHandler的例子可以参考[官方的XLSX2CVS](https://svn.apache.org/repos/asf/poi/trunk/src/examples/src/org/apache/poi/xssf/eventusermodel/XLSX2CSV.java)

##3、写在最后

Excel本身有很多已知的限制，如最大行数和最大列数(这些限制可以参考[SpreadsheetVersion](http://poi.apache.org/apidocs/org/apache/poi/ss/SpreadsheetVersion.html))，理论上只要你有足够大的内存，你就能使用Workbook API对任意Excel进行读写。

很多场景需要我们克服内存限制，总结下来有以下方案：

1、大文解析使用SAX

2、大文件写入使用SXSSFWorkbook

还有一种妥协的方案是将数据分别写入到多个Excel中，最后对这些Excel打包。

最近在Github上看到阿里的一位大佬开发的[EasyExcel](https://github.com/alibaba/easyexcel)，不过还没实际使用过，处于观望阶段...



参考链接：

https://en.wikipedia.org/wiki/Office_Open_XML

http://poi.apache.org/components/spreadsheet/how-to.html