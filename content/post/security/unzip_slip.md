---
typora-root-url: ../../../static
title: "unzip_slip漏洞"
date: 2021-07-19T17:07:36+08:00
draft: false
categories: ["security"]
---

## 漏洞代码
如下图所示，存在一个上传压缩包的接口，会对上传的文件自动进行解压：

    @RequestMapping("upload")
    public ResponseVo<Map<String,Object>> upload(MultipartFile file) throws IOException {
        // 先将上传的压缩文件进行保存
        File target = new File(basePath + File.separator + IdUtil.simpleUUID() + ".zip");
        file.transferTo(target);
        // 将上传的文件解压到basePath下，但是没有进行安全防护
        unzip(target.getAbsolutePath(), basePath,true);
        HashMap<String, Object> res = new HashMap<>();
        res.put("path",target.getPath());
        return ResultUtils.success("上传解压成功！",res);
    }

继续看到 `unzip()` 函数的具体实现：

    /**
     * 模拟不安全的解压
     * 没有检查entry的文件类型和全路径名中的路径穿越符，从而导致漏洞
     */
    public static void unzip(String zipFilePath, String targetPath,boolean overwrite) throws IOException {
        ZipFile zipFile = new ZipFile(zipFilePath,Charset.forName("GBK"));
        Enumeration<? extends ZipEntry> entryEnum = zipFile.entries();
        if (null != entryEnum) {
            while (entryEnum.hasMoreElements()) {
                ZipEntry zipEntry = entryEnum.nextElement();
                // 获取entry的全路径文件名，根据文件名进行解压保存
                // 没有考虑entry的文件类型和全路径名中的路径穿越符，从而导致漏洞
                String filePath = zipEntry.getName();
                if (zipEntry.isDirectory()) {
                    File dir = new File(targetPath + File.separator + filePath);
                    dir.mkdirs();
                } else {
                    File targetFile = new File(targetPath + File.separator + filePath);
                    if (!targetFile.exists() || overwrite) {
                        targetFile.getParentFile().mkdirs();
                        try (InputStream inputStream = zipFile.getInputStream(zipEntry);
                             FileOutputStream outputStream = new FileOutputStream(targetFile);
                             FileLock fileLock = outputStream.getChannel().tryLock()) {
                            if (null != fileLock) {
                                StreamUtils.copy(inputStream, outputStream);
                            }
                        }
                    }
                }
            }
        }
        zipFile.close();
    }

可以看到，解压处理逻辑，没有注意路径穿越的防范问题，从而导致unzip slip漏洞。

## 攻击演示
如下图所示，构造一个特殊处理的zip文件，这个zip文件的entry名是包含路径穿越符号的：

![][p1]

然后上传此压缩文件：

![][p2]

![][p3]

访问此ftl文件对应的路由，成功弹出计算器：

![][p4]

## 漏洞修复
`unzip()` 函数中，对zip文件内的entry的名字进行了判断，不能包含路径穿越符号：

    /**
     * 安全写法：检查entry的文件类型和路径穿越
     * 模拟修复：不安全的解压
     */
    public static void unzipWithSafe(String zipFilePath, String targetPath,boolean overwrite) throws IOException {
        ZipFile zipFile = new ZipFile(zipFilePath, Charset.forName("GBK"));
        Enumeration<? extends ZipEntry> entryEnum = zipFile.entries();
        if (null != entryEnum) {
            while (entryEnum.hasMoreElements()) {
                ZipEntry zipEntry = entryEnum.nextElement();
                // 获取entry的全路径文件名，根据文件名进行解压保存
                String filePath = zipEntry.getName();
                // 检查entry的文件类型和entry全路径名是否存在路径穿越
                if(filePath.contains("..")){
                    throw new RuntimeException("压缩文件存在路径穿越风险！停止上传！");
                }
                if(!zipEntry.isDirectory()&&!whiteList.contains(filePath.substring(filePath.lastIndexOf(".")+1))){
                    throw new RuntimeException("压缩文件内存在非法文件！停止上传！");
                }
                if (zipEntry.isDirectory()) {
                    File dir = new File(targetPath + File.separator + filePath);
                    dir.mkdirs();
                } else {
                    File targetFile = new File(targetPath + File.separator + filePath);
                    if (!targetFile.exists() || overwrite) {
                        targetFile.getParentFile().mkdirs();
                        try (InputStream inputStream = zipFile.getInputStream(zipEntry);
                             FileOutputStream outputStream = new FileOutputStream(targetFile);
                             FileLock fileLock = outputStream.getChannel().tryLock()) {
                            if (null != fileLock) {
                                StreamUtils.copy(inputStream, outputStream);
                            }
                        }
                    }
                }
            }
        }
        zipFile.close();
    }







[p1]:/media/2021-07-19-1.png
[p2]:/media/2021-07-19-2.png
[p3]:/media/2021-07-19-3.png
[p4]:/media/2021-07-19-4.png





