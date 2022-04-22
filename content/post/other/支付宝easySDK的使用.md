---
typora-root-url: ../../../static
title: "支付宝easySDK的使用"
date: 2020-05-11T08:04:36+08:00
draft: false
categories: ["Other"]
tags: ["支付宝"]
---

## 前言
需要知道的是，alipayEasySDK支持alipaySDK所支持的所有功能，而且alipayEasySDK提供了很多精简后的api（而在alipaySDK中使用相同的功能的api需要设置很多繁琐的参数），如果存在暂时没有提供的api，可以使用其通用接口调用支付宝的api。官方文档如下：

1. <a href='https://github.com/alipay/alipay-easysdk/blob/master/APIDoc.md' target="_blank">alipayEasySDK文档</a>
2. <a href='https://opendocs.alipay.com/apis/' target="_blank">alipayApi文档</a>

## 简单上手
### 准备资源文件
1. 准备支付宝的几个证书文件。
2. 准备填写了支付宝一些配置的配置文件。

我的支付宝配置aplipay.properties文件内容如下：

	config.protocol = http
	config.gatewayHost = openapi.alipaydev.com
	config.signType = RSA2
	config.appId = 2016102400753062
	config.notifyUrl=http://313j23614b.zicp.vip/takeExpress_war_exploded/alipay/notify
	# 为避免私钥随源码泄露，推荐从文件中读取私钥字符串而不是写入源码中
	config.merchantPrivateKey = xxxxxxxxxxx

我的有关Alipay的资源文件列表如下：

![Alipay的资源文件][p0]

### 编写GetConfig类
GetConfig类的作用是返回一个加载了alipay.properties配置的Config实例，代码如下：

	public class GetConfig {
	
	    private static Config config;
	
	    static{
	        Properties properties=new Properties();
	        //获取配置文件的path
	        String path = GetConfig.class.getClassLoader().getResource("api/alipay/alipay.properties").getPath();
	        try (FileInputStream inputStream=new FileInputStream(path);){
	            properties.load(inputStream);
	            Config configTmp = new Config();
	            configTmp.protocol=properties.getProperty("config.protocol");
	            configTmp.gatewayHost=properties.getProperty("config.gatewayHost");
	            configTmp.signType=properties.getProperty("config.signType");
	            configTmp.appId=properties.getProperty("config.appId");
	            configTmp.merchantPrivateKey=properties.getProperty("config.merchantPrivateKey");
	
	            //设置异步回调url,进行订单后续操作
	            configTmp.notifyUrl=properties.getProperty("config.notifyUrl");
	
	            //注：证书文件路径支持设置为文件系统中的路径或CLASS_PATH中的路径，优先从文件系统中加载，加载失败后会继续尝试从CLASS_PATH中加载
	            configTmp.merchantCertPath = GetConfig.class.getClassLoader().getResource("api/alipay/appCertPublicKey_2016102400753062.crt").getPath();
	            configTmp.alipayCertPath = GetConfig.class.getClassLoader().getResource("api/alipay/alipayCertPublicKey_RSA2.crt").getPath();
	            configTmp.alipayRootCertPath = GetConfig.class.getClassLoader().getResource("api/alipay/alipayRootCert.crt").getPath();
	            //注：如果采用非证书模式，则无需赋值上面的三个证书路径，改为赋值如下的支付宝公钥字符串即可
	            // config.alipayPublicKey = "<-- 请填写您的支付宝公钥，例如：MIIBIjANBg... -->";
	            config=configTmp;
	        } catch (Exception e) {
	            e.printStackTrace();
	        }
	    }
	
	    /**
	     * @return 返回alipay的一个配置实例
	     */
	    public static Config get(){
	        return config;
	    }
	}

### 编写工具类PayOnPC

此工具类主要用于PC网站支付，代码内容如下：

	package com.express.utils.alipay;
	
	import com.alipay.easysdk.factory.Factory;
	import com.alipay.easysdk.payment.facetoface.models.AlipayTradePrecreateResponse;
	import com.alipay.easysdk.util.generic.models.AlipayOpenApiGenericResponse;
	import net.sf.json.JSONObject;
	import java.util.HashMap;
	import java.util.UUID;
	
	/**
	 * 基于网站的支付
	 * @author Cassie
	 */
	public class PayOnPc {
	
	    static {
	        // 1. 设置参数（全局只需设置一次）
	        Factory.setOptions(GetConfig.get());
	    }
	
	    /**
	     * @param subject 交易订单的主题，例如：苹果6plus
	     * @param num 交易订单的订单号，在全站中此订单号需要唯一
	     * @param money 交易金额
	     * @return 一个二维码链接
	     */
	    public static JSONObject pay(String subject,String num,String money){
	        JSONObject res = new JSONObject();
	        res.put("result",false);
	        try {
	            AlipayTradePrecreateResponse response = Factory.Payment.FaceToFace().preCreate(subject, num, money);
	            res.put("result",true);
	            res.put("num",num);
	            res.put("url",response.qrCode);
	        } catch (Exception e) {
	            e.printStackTrace();
	        }
	        return res;
	    }
	
	
	    /**
	     * 提供查询支付状态的接口
	     * @param num 商家订单号
	     * @return 支付宝返回的结果
	     * @throws Exception 异常
	     */
	   public static JSONObject query(String num){
	       String body;
	       try {
	           body = Factory.Payment.Common().query(num).httpBody;
	       } catch (Exception e) {
	           JSONObject error = new JSONObject();
	           error.put("code","400");
	           error.put("msg","订单不存在");
	           return error;
	       }
	       return (JSONObject) JSONObject.fromObject(body).get("alipay_trade_query_response");
	   }
	
	
	    /**
	     * 取消未完成的订单，当主动查询发现为支付时，需要进行交易撤销
	     * @param num 订单号
	     * @return 是否撤销成功
	     */
	   public static Boolean cancelPay(String num){
	       try {
	           //alipay_trade_cancel_response
	           String body = Factory.Payment.Common().cancel(num).httpBody;
	           System.out.println(body);
	           JSONObject res = (JSONObject) JSONObject.fromObject(body).get("alipay_trade_cancel_response");
	           if(res.has("code")&&res.has("msg")){
	               String code = res.get("code").toString();
	               String msg = res.get("msg").toString();
	               return "Success".equals(msg) && "10000".equals(code);
	           }
	       } catch (Exception e) {
	           e.printStackTrace();
	       }
	       return false;
	   }
	
	
	    /**
	     * 对订单进行退款，只能全额退款
	     * @param num 订单号
	     * @param money 退款金额
	     * @return 是否退款成功
	     */
	   public static Boolean refund(String num,String money){
	       try {
	           String body = Factory.Payment.Common().refund(num, money).httpBody;
	           JSONObject res = (JSONObject) JSONObject.fromObject(body).get("alipay_trade_refund_response");
	           System.out.println(res);
	           if(res.has("code")&&res.has("msg")){
	               String code = res.get("code").toString();
	               String msg = res.get("msg").toString();
	               return "Success".equals(msg) && "10000".equals(code);
	           }
	       } catch (Exception e) {
	           e.printStackTrace();
	       }
	       return false;
	   }
	
	
	    /**
	     * 转账
	     * @param alipayLoginId 支付宝账号
	     * @param trueName 真实姓名
	     * @param money 金额
	     * @return 是否成功
	     * @throws Exception 异常
	     */
	   public static Boolean transfer(String alipayLoginId,String trueName,String money) throws Exception {
	       //构造请求参数
	       HashMap<String, Object> bizParams = new HashMap<>();
	       bizParams.put("out_biz_no",UUID.randomUUID().toString());
	       bizParams.put("trans_amount",money);
	       bizParams.put("biz_scene","DIRECT_TRANSFER");
	       bizParams.put("product_code","TRANS_ACCOUNT_NO_PWD");
	       JSONObject payeeInfo = new JSONObject();
	       payeeInfo.put("identity",alipayLoginId);
	       payeeInfo.put("identity_type","ALIPAY_LOGON_ID");
	       payeeInfo.put("name",trueName);
	       bizParams.put("payee_info",payeeInfo.toString());
	       bizParams.put("order_title","快递代取佣金");
	       AlipayOpenApiGenericResponse response = Factory.Util.Generic().execute("alipay.fund.trans.uni.transfer", null, bizParams);
	       return "10000".equals(response.code)&&"Success".equals(response.msg);
	   }
	}







[p0]:/media/20200511-1.png