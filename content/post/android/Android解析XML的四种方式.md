---
typora-root-url: ../../../static
title: "Android解析XML的四种方式"
date: 2019-09-26T20:14:36+08:00
lastmod: 2019-09-28T21:28:36+08:00
draft: false
categories: ["Android"]
tags: ["java解析XML"]
---

## DOM方式解析XML
DOM方式解析XML是将XML先全部加载到内存当中，所以 **适用于较小的XML文件，可随机访问，可读可写** 。DOM方式解析逻辑如下：

1. 利用DOM对象树解析器工厂（ **DocumentBuilderFactory** ）生成DOM对象树解析器（ **DocumentBuilder** ）。
2. 利用DOM对象树解析器将给定的 **文件或输入流** 解析成XML文档，并返回一个 **Document** 对象。
3. 通过Document对象的 `getDocumentElement()` 方法获取XML文档的文档元素（每个XML文档有且仅有一个文档元素）。
4. 通过文档元素获取元素下的子节点集合，调用文档元素的 `getElementsByTagName()` 方法。
5. 遍历子节点集合。

一个案例如下：

①XML文档如下：

	<?xml version="1.0" encoding="UTF-8"?>
	<students>
	    <student id="1">
	        <name>pinger</name>
	        <age>20</age>
	    </student>
	    <student id="2">
	        <name>菜</name>
	        <age>21</age>
	    </student>
	    <student id="3">
	        <name>杨胖</name>
	        <age>20</age>
	    </student>
	</students>

②解析代码如下(此代码写在一个Activity里，获取的是res/raw下的xml文件)：

	InputStream inputStream =(InputStream)getResources().openRawResource(R.raw.students);
	DocumentBuilderFactory factory=DocumentBuilderFactory.newInstance();
	DocumentBuilder builder=factory.newDocumentBuilder();
	Document document=builder.parse(inputStream);
	//获取students元素节点
	Element studentsNode=document.getDocumentElement();
	//获取所有的student节点
	NodeList studentNodeList=studentsNode.getElementsByTagName("student");
	for(int i=0;i<studentNodeList.getLength();i++){
	    //获取student节点下的name节点和age节点，并且读取这两个节点的内容
	    Element currentElement=(Element)studentNodeList.item(i);
	    Node nameNode=currentElement.getElementsByTagName("name").item(0);
	    Node ageNode=currentElement.getElementsByTagName("age").item(0);
	    int id=new Integer(currentElement.getAttribute("id"));
	    String name=nameNode.getTextContent();
	    int age=Integer.parseInt(ageNode.getTextContent());
	    System.out.println("id:"+id+",name:"+name+",age:"+age);
	}

③解析效果如下：

![DOM方式解析结果][p0]

## SAX方式解析XML
SAX的全称是： **Simple API for XML** ,属于 **事件驱动** 型解析，消耗资源少，适合解析较大的XML文件， **只能读不能写，顺序访问，不能随机访问** 。对于事件源，就是给定的XML文档，事件处理器需要自己写。此方法的一般步骤为：

1. 通过继承org.xml.sax.helpers.DefaultHandler编写自己的事件处理器。
2. 重写自己写的事件处理器的四个方法（看下面的代码便知）。
3. 实例化一个SAX解析器工厂（SAXParserFactory.newInstance()方式）。
4. 通过解析器工厂获取SAX解析器（SAXParser）。
5. 实例化自己写的事件处理器（Handler）对象。
6. 通过SAX解析器获取事件源以及设置事件处理器。
7. 事件处理器自动获取事件，执行事件处理逻辑。
8. 通过事件处理器对象的某些方法（自己写）获取处理逻辑的结果。

一个简单的案例（就事件处理器编写以及一些注意事项而言）如下：

①XML文档如下：

![XML文档说明][p2]

②StudentHandler的代码如下：

	public class StudentHandler extends DefaultHandler {
	    @Override
	    public void startDocument() throws SAXException {
	        //这个不是开始加载文档元素，而是开始加载XML的第一行
	        System.out.println("文档开始加载。。。。。");
	    }
	
	    @Override
	    public void endDocument() throws SAXException {
	        System.out.println("文档结束加载。。。。。");
	    }
	
	    @Override
	    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
	        /*
	            uri - 命名空间URI，如果该元素没有命名空间URI或未执行命名空间处理，则为空字符串。
	            localName - 本地名称（无前缀），或空字符串，如果未执行命名空间处理。
	            qName - 限定名称（带前缀），如果限定名称不可用，则为空字符串。
	         */
	        System.out.println("元素节点开始加载："+uri+"，"+localName+"，"+qName+"，"+attributes.getValue("id"));
	    }
	
	    @Override
	    public void endElement(String uri, String localName, String qName) throws SAXException {
	        System.out.println("元素节点结束加载："+uri+"，"+localName+"，"+qName);
	    }
	
	    @Override
	    public void characters(char[] ch, int start, int length) throws SAXException {
	        System.out.println("扫描文本节点："+new String(ch,start,length));
	    }
	}

③加载事件源以及设置事件处理器的关键代码如下：

    //通过SAX方式读取XML
    SAXParserFactory factory=SAXParserFactory.newInstance();
    //获取SAX解析器
    SAXParser parser=factory.newSAXParser();
    //new一个事件处理器
    StudentHandler handler=new StudentHandler();
    //加载数据源和设置事件处理器
    parser.parse(getResources().openRawResource(R.raw.students),handler);

④部分运行结果如下：

![SAX方式解析结果1][p1]

⑤一个完整的解析XML数据到Student实体的StudentHandler代码如下：

	public class StudentHandler extends DefaultHandler {
	    private List<Student> students;
	    private Student student;
	    private String prevTag;
	
	    @Override
	    public void startDocument() throws SAXException {
	        //初始化students
	        students=new ArrayList<>();
	    }
	
	    @Override
	    public void endDocument() throws SAXException {
	        //do nothing
	    }
	
	    @Override
	    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
	        /*
	            uri - 命名空间URI，如果该元素没有命名空间URI或未执行命名空间处理，则为空字符串。
	            localName - 本地名称（无前缀），或空字符串，如果未执行命名空间处理。
	            qName - 限定名称（带前缀），如果限定名称不可用，则为空字符串。
	         */
	        //如果当前节点是student元素节点
	        if("student".equals(localName)){
	            student=new Student();
	            //将student元素节点的id属性赋值给一个student对象的id
	            student.setId(Integer.parseInt(attributes.getValue("id")));
	        }
	        prevTag=localName;
	    }
	
	    @Override
	    public void endElement(String uri, String localName, String qName) throws SAXException {
	        if("student".equals(localName)){
	            //解析完一个student元素后，将得到的student加入students集合
	            students.add(student);
	        }
	        //必须在元素节点结束后将prevTag清空，为了防止后面给student的属性赋值的时候出现多次赋值从而被空白文本节点覆盖
	        prevTag=null;
	    }
	
	    @Override
	    public void characters(char[] ch, int start, int length) throws SAXException {
	        if(prevTag!=null){
	            String content=new String(ch,start,length).trim();
	            if("name".equals(prevTag)){
	                //解析到了name元素节里的文本节点时
	                student.setName(content);
	            }else if("age".equals(prevTag)){
	                //解析到了age元素节里的文本节点时
	                student.setAge(Integer.parseInt(content));
	            }
	        }
	    }
	
	    //一个获取students集合的方法
	    public List<Student> getStudents(){
	        return this.students;
	    }
	}

⑥测试StudentHandler的代码如下：

    //读取res/xml下的xml文件
    try{
        //通过SAX方式读取XML
        SAXParserFactory factory=SAXParserFactory.newInstance();
        //获取SAX解析器
        SAXParser parser=factory.newSAXParser();
        //new一个事件处理器
        StudentHandler handler=new StudentHandler();
        //加载数据源和设置事件处理器
        parser.parse(getResources().openRawResource(R.raw.students),handler);
        for(Student v:handler.getStudents()){
            System.out.println(v);
        }

⑦测试结果如下：

![SAX解析XML结果2][p3]

## PULL方式解析XML
PULL方式解析XML在Android用的比较多，而且此方式被继承到了Android。此种解析方式也是基于 **事件驱动** ，与SAX方式相似，但是不是自动提取事件，而是需要自己主动提取事件，并根据提取的事件类型进行相应的事件处理逻辑。此方法的一般步骤如下：

1. 通过Resource类对象的getXml()方法返回一个XmlResourceParser对象。
2. 通过XmlResourceParser对象的事件状态进行对应的事件处理逻辑。

一个案例如下：

①XML文档如下：

	<?xml version="1.0" encoding="UTF-8"?>
	<students>
	    <student id="1">
	        <name>P1n93r</name>
	        <age>18</age>
	    </student>
	    <student id="2">
	        <name>菜</name>
	        <age>19</age>
	    </student>
	    <student id="3">
	        <name>杨胖</name>
	        <age>18</age>
	    </student>
	</students>

①工具类XmlParserForStudent代码如下：

	public class XmlParserForStudent {
	    public static List<Student> getStudents(XmlResourceParser parser) throws XmlPullParserException, IOException {
	        List<Student> students =null;
	        Student student=null;
	        String preTag = null;
	
	        //获取事件的类型
	        /*
	            int START_DOCUMENT = 0;
	            int END_DOCUMENT = 1;
	            int START_TAG = 2;
	            int END_TAG = 3;
	            int TEXT = 4;
	         */
	        //Resource解析器的首次默认光标是START_DOCUMENT，当执行一次next操作后，光标还是在START_DOCUMENT
	        //所以我们给一个不是START_DOCUMENT的光标初始值，防止students实例化两次
	        int eventType=-1;
	        //开始解析
	        while(eventType!=XmlResourceParser.END_DOCUMENT){
	            switch (eventType) {
	                case XmlResourceParser.START_DOCUMENT:
	                    //开始解析文档时，实例化students
	                    students = new ArrayList<>();
	                    break;
	                case XmlResourceParser.START_TAG:
	                    preTag = parser.getName();
	                    if ("student".equals(preTag)) {
	                        //如果是student元素节点，则实例化一个Student对象
	                        student = new Student();
	                        student.setId(Integer.parseInt(parser.getAttributeValue(null, "id")));
	                    }
	                    break;
	                case XmlResourceParser.TEXT:
	                    if (preTag != null) {
	                        String content = parser.getText();
	                        if ("name".equals(preTag)) {
	                            student.setName(content);
	                        } else if ("age".equals(preTag)) {
	                            student.setAge(Integer.parseInt(content));
	                        }
	                    }
	                    break;
	                case XmlResourceParser.END_TAG:
	                    String currentTag=parser.getName();
	                    if ("student".equals(currentTag)) {
	                        students.add(student);
	                    }
	                    preTag = null;
	                    break;
	                default:
	                    break;
	            }
	            //将解析器光标移动到下一步
	            eventType=parser.next();
	        }
	        return students;
	    }
	}

②解析结果如下图所示：

![PULL方式解析XML结果1][p4]

## DOM4J方式解析XML
此种解析方式性能优异，推荐使用此种方式进行XML解析。一般的操作步骤如下：

1. 通过SAXReader或者DomReader构建DOM4J树对象。
2. 获取XML文档的根节点（通过DOM4J树对象的getRootElement()方法实现）。
3. 获取根节点的子节点或字节点集合（org.dom4j.Element对象的element()或者elements()方法）。
4. 获取子节点的属性或者其文本（org.dom4j.Element对象的attribute()方法获得属性对象，getText()方法获得其文本内容）。

一个案例如下：

①XML文档如下：

	<students>
	    <student id="1">
	        <name>pinger</name>
	        <age>3</age>
	    </student>
	    <student id="2">
	        <name>4</name>
	        <age>21</age>
	    </student>
	    <student id="3">
	        <name>杨胖</name>
	        <age>3</age>
	    </student>
	</students>

②DOM4J方式解析XML关键代码如下：

    //DomReader方式获取dom4j树
    //此种方式只能将一个现有的w3c DOM树构建成dom4j树
    DocumentBuilderFactory factory=DocumentBuilderFactory.newInstance();
    DocumentBuilder builder=factory.newDocumentBuilder();
    org.w3c.dom.Document w3cDocument=builder.parse(getResources().getAssets().open("students.xml"));
    DOMReader reader = new DOMReader();
    //此Document对象不是org.w3c.dom.Document,而是org.dom4j.Document
    Document document = reader.read(w3cDocument);

    //获取根节点
    Element root=document.getRootElement();
    System.out.println("根节点为："+root.getName());
    List<Element> studentNodes=root.elements("student");
    for(Element v:studentNodes){
        int id=Integer.parseInt(v.attribute("id").getValue());
        String name=v.element("name").getText();
        int age=Integer.parseInt(v.element("age").getText());
        System.out.println(name+"，"+age);
    }

③解析结果如下图所示：

![DOM4J方式解析XML结果][p5]

## 结言
DOM方式只适合解析较小的XML文件；SAX和PULL方式使用方法类似，但是由于不能随机访问，且只读不可写。 **DOM4J方式性能很好，又灵活简单，最推荐使用** 。



[p0]:/media/20190928-1.png
[p1]:/media/20190928-2.png
[p2]:/media/20190928-3.png
[p3]:/media/20190928-4.png
[p4]:/media/20190928-5.png
[p5]:/media/20190928-6.png