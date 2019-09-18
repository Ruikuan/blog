# https 发布企业证书签名的 ios App

之前的 p12 文件和 mobileprovision 文件的不详细讲了。总之就是得到这两个文件，将 p12 文件导入 KeyChain，导入时要输入密码。然后项目里面设置用它签名。然后打包，然后导出 ipa。

下面主要是记录一下发布到 https 的部分。

## bp.plist 

生成对应的 bp.plist 文件，内容如下

```xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
   <!-- array of downloads. -->
   <key>items</key>
   <array>
       <dict>
           <!-- an array of assets to download -->
           <key>assets</key>
           <array>
               <!-- software-package: the ipa to install. -->
               <dict>
                   <!-- required.  the asset kind. -->
                   <key>kind</key>
                   <string>software-package</string>
                   <key>url</key>
                   <string>https://example.com/app.ipa</string>
               </dict>
               <!-- display-image: the icon to display during download. -->
               <dict>
                   <key>kind</key>
                   <string>display-image</string>
                   <!-- optional. icon needs shine effect applied. -->
                   <key>needs-shine</key>
                   <true/>
                   <key>url</key>
                   <string>https://example.com/logo.png</string>
               </dict>
               <!-- full-size-image: the large 512×512 icon used by iTunes. -->
               <dict>
                   <key>kind</key>
                   <string>full-size-image</string>
                              <!-- optional. icon needs shine effect applied. -->
                   <key>needs-shine</key>
                   <true/>
                   <key>url</key>
                   <string>https://example.com/logo.png</string>
               </dict>
           </array><key>metadata</key>
           <dict>
               <!-- required -->
               <key>bundle-identifier</key>
               <string>com.xx.awsomeapp</string>
               <!-- optional (software only) -->
               <key>bundle-version</key>
               <string>1.0</string>
               <!-- required. the download kind. -->
               <key>kind</key>
               <string>software</string>
               <!-- optional. displayed during download; -->
               <!-- typically company name -->
               <key>subtitle</key>
               <string>Apple</string>
               <!-- required. the title to display during the download. -->
               <key>title</key>
               <string>Very NB App</string>
           </dict>
       </dict>
   </array>
</dict>
</plist>
```

### 生成浏览器安装链接

```
<a href="itms-services://?action=download-manifest&url=https://example.com/bp.plist">click to install</a>
```

用 safari 浏览网页点击就可以安装了。安装运行可能会提示不信任发布者，到设置——通用——设备管理 里面信任对应账号即可。