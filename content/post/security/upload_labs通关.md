---
typora-root-url: ../../../static
title: "upload_labs通关"
date: 2020-11-08T13:04:36+08:00
draft: false
tags: ["file_upload"]
categories: ["security"]
---

## 第一关
### 源码
	function checkFile() {
	    var file = document.getElementsByName('upload_file')[0].value;
	    if (file == null || file == "") {
	        alert("请选择要上传的文件!");
	        return false;
	    }
	    //定义允许上传的文件类型
	    var allow_ext = ".jpg|.png|.gif";
	    //提取上传文件的类型
	    var ext_name = file.substring(file.lastIndexOf("."));
	    //判断上传文件类型是否允许上传
	    if (allow_ext.indexOf(ext_name + "|") == -1) {
	        var errMsg = "该文件不允许上传，请上传" + allow_ext + "类型的文件,当前文件类型为：" + ext_name;
	        alert(errMsg);
	        return false;
	    }
	}

### 过关
第一关是一个 **前端校验** ，直接将php文件的后缀改成 `.jpg` ，然后点击上传后抓包，直接改包，此时将文件名再改为 `.php` 后缀，然后放包即可上传。发送的数据包以及回包如下：

![第一关][p0]

## 第二关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        if (($_FILES['upload_file']['type'] == 'image/jpeg') || ($_FILES['upload_file']['type'] == 'image/png') || ($_FILES['upload_file']['type'] == 'image/gif')) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH . '/' . $_FILES['upload_file']['name']            
	            if (move_uploaded_file($temp_file, $img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '文件类型不正确，请重新上传！';
	        }
	    } else {
	        $msg = UPLOAD_PATH.'文件夹不存在,请手工创建！';
	    }
	}

### 过关
由源码知道，就是校验了请求包的content-type，所以我们可以直接上传一个php文件，然后修改其content-type为 `image/jpeg` 即可绕过校验。如下：

![第二关][p1]

## 第三关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array('.asp','.aspx','.php','.jsp');
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_name = deldot($file_name);//删除文件名末尾的点
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = strtolower($file_ext); //转换为小写
	        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
	        $file_ext = trim($file_ext); //收尾去空
	
	        if(!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;            
	            if (move_uploaded_file($temp_file,$img_path)) {
	                 $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '不允许上传.asp,.aspx,.php,.jsp后缀文件！';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
这一关无法使用 `0x00/%00` 截断，无法使用 `::$DATA` 或者 `.` 绕过，也无法使用大小写绕过。思路就是上传apache可解析的其他类型文件，比如 `phtml` 等。

这里举一个实际案例，上次挖到一个任意文件上传，但是是springboot项目，且项目没有配置解析 `jsp` ，上传了 `jsp` 文件并不解析。后来发现项目使用了freemaker，于是上传 `ftl` 文件即可解析，于是可以执行任何freemaker语句。这一关的通关过程如下：

![第三关][p2]

解析效果如下(前提是apache配置的可以解析phtml文件嗷)：

![第三关解析效果][p3]

## 第四关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".php1",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".pHp1",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".ini");
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_name = deldot($file_name);//删除文件名末尾的点
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = strtolower($file_ext); //转换为小写
	        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
	        $file_ext = trim($file_ext); //收尾去空
	
	        if (!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.$file_name;
	            if (move_uploaded_file($temp_file, $img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '此文件不允许上传!';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
这一关可以利用 `.` 绕过， `deldot($file_name)` 可以去掉末尾的点，那么我们上传的文件名如果为 `index.php. .` ，去掉最后一个点 `.` 后，结果文件名会是 `index.php. ` ，末尾有一个点+空格，最终可以这样绕过的原因是：文件上传的路径拼接了上传文件的文件名（windows下文件上传，最后的点和空格会被去掉）。如果是拼接的后缀，则无法利用（因为此时后缀为一个空格）。完整的过程如下：

![第四关0][p4]

访问upload/index.php，效果如下：

![第四关1][p5]

## 第五关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_name = deldot($file_name);//删除文件名末尾的点
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = strtolower($file_ext); //转换为小写
	        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
	        $file_ext = trim($file_ext); //首尾去空
	        
	        if (!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.$file_name;
	            if (move_uploaded_file($temp_file, $img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '此文件类型不允许上传！';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
这一关也可以用第四关的方法，而且目前只有第四关的方法能过，就不再演示啦。其实第四关还有一种方法（这一关不能，因为添加了黑名单），就是上传 `.htaccess` 文件，然后在这个文件中加一条解析规则，让 `png` 等其他格式的文件能解析为php。从而上传一个 `png` 文件，但是内容是php木马，则可以访问这个 `png` 文件来执行木马啦。 `.htaccess` 添加解析规则的示例如下：

	// 方式一，精确的匹配某个文件
	<FilesMatch "文件名">
	  SetHandler application/x-httpd-php
	</FilesMatch>

	//方式二，匹配某种类型
	AddType application/x-httpd-php .jpg

## 第六关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_name = deldot($file_name);//删除文件名末尾的点
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
	        $file_ext = trim($file_ext); //首尾去空
	
	        if (!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
	            if (move_uploaded_file($temp_file, $img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '此文件类型不允许上传！';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
这一关不能用第四关和四五关的方法了，但是发现没有做大小写处理，所以直接大写绕过：

![第六关][p6]

## 第七关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
	        $file_name = $_FILES['upload_file']['name'];
	        $file_name = deldot($file_name);//删除文件名末尾的点
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = strtolower($file_ext); //转换为小写
	        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
	        
	        if (!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
	            if (move_uploaded_file($temp_file,$img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '此文件不允许上传';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
通过阅读源码可以知道，没有去掉文件名的前后空格，所以可以利用空格绕过，上传的文件的文件名末尾加一个空格即可，如下：

![第七关][p7]

## 第八关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = strtolower($file_ext); //转换为小写
	        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
	        $file_ext = trim($file_ext); //首尾去空
	        
	        if (!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.$file_name;
	            if (move_uploaded_file($temp_file, $img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '此文件类型不允许上传！';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
根据源码，可以发现，没有去除上传文件名末尾的点，所以可以利用 `.` 绕过（当然，也可以用第四关的方法）。如下：

![第八关][p8]

## 第九关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_name = deldot($file_name);//删除文件名末尾的点
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = strtolower($file_ext); //转换为小写
	        $file_ext = trim($file_ext); //首尾去空
	        
	        if (!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
	            if (move_uploaded_file($temp_file, $img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '此文件类型不允许上传！';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
利用windos特性，可以使用 `::$DATA` 绕过， `::$DATA` 之后的字符在windows下会被认为是文件流数据，且保持 `::$DATA` 之前的文件名。如下：

![第九关][p9]

## 第十关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_name = deldot($file_name);//删除文件名末尾的点
	        $file_ext = strrchr($file_name, '.');
	        $file_ext = strtolower($file_ext); //转换为小写
	        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
	        $file_ext = trim($file_ext); //首尾去空
	        
	        if (!in_array($file_ext, $deny_ext)) {
	            $temp_file = $_FILES['upload_file']['tmp_name'];
	            $img_path = UPLOAD_PATH.'/'.$file_name;
	            if (move_uploaded_file($temp_file, $img_path)) {
	                $is_upload = true;
	            } else {
	                $msg = '上传出错！';
	            }
	        } else {
	            $msg = '此文件类型不允许上传！';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
还是利用第四关的方法，如下：

![第十关][p10]

## 第十一关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])) {
	    if (file_exists(UPLOAD_PATH)) {
	        $deny_ext = array("php","php5","php4","php3","php2","html","htm","phtml","pht","jsp","jspa","jspx","jsw","jsv","jspf","jtml","asp","aspx","asa","asax","ascx","ashx","asmx","cer","swf","htaccess","ini");
	
	        $file_name = trim($_FILES['upload_file']['name']);
	        $file_name = str_ireplace($deny_ext,"", $file_name);
	        $temp_file = $_FILES['upload_file']['tmp_name'];
	        $img_path = UPLOAD_PATH.'/'.$file_name;        
	        if (move_uploaded_file($temp_file, $img_path)) {
	            $is_upload = true;
	        } else {
	            $msg = '上传出错！';
	        }
	    } else {
	        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
	    }
	}

### 过关
首先看源码，获取上传文件的文件名，然后根据黑名单将文件名中的黑名单字符替换成空，于是想到可以通过双写绕过，所谓双写不是类似 `.phpphp` 这种，这样两个 `php` 字符都会被替换成空，不能达到效果。可以这样双写 `.phphpp` ，这个字符串中间的 `php` 被替换成空，替换后就是 `.php` 了，于是可以达到效果，如下：

![第十一关][p11]

## 第十二关
### 源码
	$is_upload = false;
	$msg = null;
	if(isset($_POST['submit'])){
	    $ext_arr = array('jpg','png','gif');
	    $file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
	    if(in_array($file_ext,$ext_arr)){
	        $temp_file = $_FILES['upload_file']['tmp_name'];
	        $img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;
	
	        if(move_uploaded_file($temp_file,$img_path)){
	            $is_upload = true;
	        } else {
	            $msg = '上传出错！';
	        }
	    } else{
	        $msg = "只允许上传.jpg|.png|.gif类型文件！";
	    }
	}

### 过关
使用了白名单，但是注意到这行代码： `$img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;` ，拼接了前端传来的参数save_path,可以想到使用%00截断，这样 `."/".rand(10, 99).date("YmdHis").".".$file_ext;` 被截断，最终生成的文件路径和文件名都是由%00前的字符串来控制。

但是这关%00截断要成功需要一定条件，如下条件：

1. php版本小于5.3.4
2. php的magic_quotes_gpc为OFF状态,如果不修改将无法上传成功，默认为ON

所以在保证上面两个条件下，复现如下：

![第十二关][p12]

## 第十三关
### 源码
	$is_upload = false;
	$msg = null;
	if(isset($_POST['submit'])){
	    $ext_arr = array('jpg','png','gif');
	    $file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
	    if(in_array($file_ext,$ext_arr)){
	        $temp_file = $_FILES['upload_file']['tmp_name'];
	        $img_path = $_POST['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;
	
	        if(move_uploaded_file($temp_file,$img_path)){
	            $is_upload = true;
	        } else {
	            $msg = "上传失败";
	        }
	    } else {
	        $msg = "只允许上传.jpg|.png|.gif类型文件！";
	    }
	}

### 过关
也是00截断，不过是post里的00截断，如下：

![十三关][p13]

## 第十四关
### 源码
	function getReailFileType($filename){
	    $file = fopen($filename, "rb");
	    $bin = fread($file, 2); //只读2字节
	    fclose($file);
	    $strInfo = @unpack("C2chars", $bin);    
	    $typeCode = intval($strInfo['chars1'].$strInfo['chars2']);    
	    $fileType = '';    
	    switch($typeCode){      
	        case 255216:            
	            $fileType = 'jpg';
	            break;
	        case 13780:            
	            $fileType = 'png';
	            break;        
	        case 7173:            
	            $fileType = 'gif';
	            break;
	        default:            
	            $fileType = 'unknown';
	        }    
	        return $fileType;
	}
	
	$is_upload = false;
	$msg = null;
	if(isset($_POST['submit'])){
	    $temp_file = $_FILES['upload_file']['tmp_name'];
	    $file_type = getReailFileType($temp_file);
	
	    if($file_type == 'unknown'){
	        $msg = "文件未知，上传失败！";
	    }else{
	        $img_path = UPLOAD_PATH."/".rand(10, 99).date("YmdHis").".".$file_type;
	        if(move_uploaded_file($temp_file,$img_path)){
	            $is_upload = true;
	        } else {
	            $msg = "上传出错！";
	        }
	    }
	}

### 过关
根据源码，可以知道程序是提取文件中二进制数据的头两个字节（魔数）来判断文件的类型，由此可以制作图片木马然后上传，制作图片木马的步骤如下：

1. 创建一个正常的图片：1.png
2. 创建一个木马文件：1.php
3. 当前目录下cmd运行：copy 1.png/b+1.php/a 2.png，输出的2.png就是图片马，可执行木马语句在图片二进制数据的最后。

上传过程如下：

![第十四关][p14]

利用文件包含访问图片马效果如下：

![第十四关1][p15]

## 第十五关
### 源码
	function isImage($filename){
	    $types = '.jpeg|.png|.gif';
	    if(file_exists($filename)){
	        $info = getimagesize($filename);
	        $ext = image_type_to_extension($info[2]);
	        if(stripos($types,$ext)>=0){
	            return $ext;
	        }else{
	            return false;
	        }
	    }else{
	        return false;
	    }
	}
	
	$is_upload = false;
	$msg = null;
	if(isset($_POST['submit'])){
	    $temp_file = $_FILES['upload_file']['tmp_name'];
	    $res = isImage($temp_file);
	    if(!$res){
	        $msg = "文件未知，上传失败！";
	    }else{
	        $img_path = UPLOAD_PATH."/".rand(10, 99).date("YmdHis").$res;
	        if(move_uploaded_file($temp_file,$img_path)){
	            $is_upload = true;
	        } else {
	            $msg = "上传出错！";
	        }
	    }
	}

### 过关
 `getimagesize()` 函数用于获取图像大小及相关信息，成功返回一个数组，失败则返回 `FALSE` 并产生一条 `E_WARNING` 级的错误信息。所以上传图片马即可，可执行语句放在图片马最后。

## 第十六关
### 源码
	function isImage($filename){
	    //需要开启php_exif模块
	    $image_type = exif_imagetype($filename);
	    switch ($image_type) {
	        case IMAGETYPE_GIF:
	            return "gif";
	            break;
	        case IMAGETYPE_JPEG:
	            return "jpg";
	            break;
	        case IMAGETYPE_PNG:
	            return "png";
	            break;    
	        default:
	            return false;
	            break;
	    }
	}
	
	$is_upload = false;
	$msg = null;
	if(isset($_POST['submit'])){
	    $temp_file = $_FILES['upload_file']['tmp_name'];
	    $res = isImage($temp_file);
	    if(!$res){
	        $msg = "文件未知，上传失败！";
	    }else{
	        $img_path = UPLOAD_PATH."/".rand(10, 99).date("YmdHis").".".$res;
	        if(move_uploaded_file($temp_file,$img_path)){
	            $is_upload = true;
	        } else {
	            $msg = "上传出错！";
	        }
	    }
	}

### 过关
exif_imagetype()读取一个图像的第一个字节并检查其签名。所以从原理上来将，也是直接上传图片马，将执行语句放在图片马的最后。

## 第十七关
### 源码
	$is_upload = false;
	$msg = null;
	if (isset($_POST['submit'])){
	    // 获得上传文件的基本信息，文件名，类型，大小，临时文件路径
	    $filename = $_FILES['upload_file']['name'];
	    $filetype = $_FILES['upload_file']['type'];
	    $tmpname = $_FILES['upload_file']['tmp_name'];
	
	    $target_path=UPLOAD_PATH.'/'.basename($filename);
	
	    // 获得上传文件的扩展名
	    $fileext= substr(strrchr($filename,"."),1);
	
	    //判断文件后缀与类型，合法才进行上传操作
	    if(($fileext == "jpg") && ($filetype=="image/jpeg")){
	        if(move_uploaded_file($tmpname,$target_path)){
	            //使用上传的图片生成新的图片
	            $im = imagecreatefromjpeg($target_path);
	
	            if($im == false){
	                $msg = "该文件不是jpg格式的图片！";
	                @unlink($target_path);
	            }else{
	                //给新图片指定文件名
	                srand(time());
	                $newfilename = strval(rand()).".jpg";
	                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
	                $img_path = UPLOAD_PATH.'/'.$newfilename;
	                imagejpeg($im,$img_path);
	                @unlink($target_path);
	                $is_upload = true;
	            }
	        } else {
	            $msg = "上传出错！";
	        }
	
	    }else if(($fileext == "png") && ($filetype=="image/png")){
	        if(move_uploaded_file($tmpname,$target_path)){
	            //使用上传的图片生成新的图片
	            $im = imagecreatefrompng($target_path);
	
	            if($im == false){
	                $msg = "该文件不是png格式的图片！";
	                @unlink($target_path);
	            }else{
	                 //给新图片指定文件名
	                srand(time());
	                $newfilename = strval(rand()).".png";
	                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
	                $img_path = UPLOAD_PATH.'/'.$newfilename;
	                imagepng($im,$img_path);
	
	                @unlink($target_path);
	                $is_upload = true;               
	            }
	        } else {
	            $msg = "上传出错！";
	        }
	
	    }else if(($fileext == "gif") && ($filetype=="image/gif")){
	        if(move_uploaded_file($tmpname,$target_path)){
	            //使用上传的图片生成新的图片
	            $im = imagecreatefromgif($target_path);
	            if($im == false){
	                $msg = "该文件不是gif格式的图片！";
	                @unlink($target_path);
	            }else{
	                //给新图片指定文件名
	                srand(time());
	                $newfilename = strval(rand()).".gif";
	                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
	                $img_path = UPLOAD_PATH.'/'.$newfilename;
	                imagegif($im,$img_path);
	
	                @unlink($target_path);
	                $is_upload = true;
	            }
	        } else {
	            $msg = "上传出错！";
	        }
	    }else{
	        $msg = "只允许上传后缀为.jpg|.png|.gif的图片文件！";
	    }
	}

### 过关
这一关有两种思路，如下：

1. 利用条件竞争，在文件未删除前访问上传的文件。
2. 对比图片 `imagecreatefrompng()` 前后的二进制数据，然后将不变的数据替换成同等长度的可执行语句。那么经过 `imagecreatefrompng()` 生成的图片仍旧会保留木马。

## 第十八关
### 源码
	$is_upload = false;
	$msg = null;
	
	if(isset($_POST['submit'])){
	    $ext_arr = array('jpg','png','gif');
	    $file_name = $_FILES['upload_file']['name'];
	    $temp_file = $_FILES['upload_file']['tmp_name'];
	    $file_ext = substr($file_name,strrpos($file_name,".")+1);
	    $upload_file = UPLOAD_PATH . '/' . $file_name;
	
	    if(move_uploaded_file($temp_file, $upload_file)){
	        if(in_array($file_ext,$ext_arr)){
	             $img_path = UPLOAD_PATH . '/'. rand(10, 99).date("YmdHis").".".$file_ext;
	             rename($upload_file, $img_path);
	             $is_upload = true;
	        }else{
	            $msg = "只允许上传.jpg|.png|.gif类型文件！";
	            unlink($upload_file);
	        }
	    }else{
	        $msg = '上传出错！';
	    }
	}

### 过关
仍旧是条件竞争。








[p0]:/media/2020-11-08-1.png
[p1]:/media/2020-11-08-2.png
[p2]:/media/2020-11-08-3.png
[p3]:/media/2020-11-08-4.png
[p4]:/media/2020-11-08-5.png
[p5]:/media/2020-11-08-6.png
[p6]:/media/2020-11-08-7.png
[p7]:/media/2020-11-08-8.png
[p8]:/media/2020-11-08-9.png
[p9]:/media/2020-11-08-10.png
[p10]:/media/2020-11-08-11.png
[p11]:/media/2020-11-08-12.png
[p12]:/media/2020-11-08-13.png
[p13]:/media/2020-11-08-14.png
[p14]:/media/2020-11-08-15.png
[p15]:/media/2020-11-08-16.png