# Xcode8 iOS自动打包（环境、证书、描述文件），更换ipa签名

感谢：[webfrogs](https://github.com/webfrogs/xcode_shell) [庞海礁](http://www.olinone.com/?p=198) [heyuan110](https://github.com/heyuan110/BashShell)

### iOS自动打包
#### 1.直接上重点。完整的打包命令行语句

```
build_cmd=${build_cmd}' -workspace '${build_workspace}'-scheme '${build_scheme}' -sdk iphoneos -configuration '${build_config}' build CODE_SIGN_IDENTITY='${code_sign_identity}' PROVISIONING_PROFILE='${provisioning_profile}' ONLY_ACTIVE_ARCH=NO -archivePath '${xcarchive_path}' archive'
```

解释一下重点参数
`-workspace`打workspace必备的参数配套使用的`-scheme`必须填写
`-sdk`使用什么sdk打包，我们这里打iphone填写`iphoneos`
`-configuration`打什么版本的包有`Release`、`Debug`、`Distribute`

重点来了打包涉及到切换证书和描述文件打不同的包。
跟上参数`build`设置这些东西。
`CODE_SIGN_IDENTITY`设置证书这里说明一下证书填写的是钥匙串里面的证书的`SHA1`值，证书的`SHA1`在钥匙串->某一个证书->右键->查看简介->滚到最下面就看到了
![](http://ww1.sinaimg.cn/large/6a23c4cdgw1f9n1titjfmj20ma034mxo.jpg)

`PROVISIONING_PROFILE`参数填写描述文件的名字，这个可以在xcode里面访问路径 xcode->最上面菜单->xcode preferences->Account->选择team选择view details->找到需要的provisioning profiles->右键show in finder `只需要名字不要后缀名`
![](http://ww2.sinaimg.cn/large/6a23c4cdgw1f9n1u257klj20aq03aglx.jpg)

`ONLY_ACTIVE_ARCH`打包包含的设备类型NO表示所有的设备

因为后面要使用`xcodebuild -exportArchive`导出包所以这里要先设置`-archivePath`路径，打包最后结尾写上`archive`打archive的包

#### 2.导出ipa包

```
xcodebuild -exportArchive -archivePath ${xcarchive_path} -exportPath ${build_path}/ipa-build -exportOptionsPlist ExportOptions.plist
```
导出ipa包3个点：
1. 设置archivePath路径，在打包的时候设置的路径
2. 导出ipa包的路径
3. 导出的ipa包有一些参数可以通过`ExportOptions.plist`文件设置，plist文件里面可以设置的参数有很多

```
Available keys for -exportOptionsPlist:
	compileBitcode : Bool
		For non-App Store exports, should Xcode re-compile the app from bitcode? Defaults to YES.
	embedOnDemandResourcesAssetPacksInBundle : Bool
		For non-App Store exports, if the app uses On Demand Resources and this is YES, asset packs are embedded in the app bundle so that the app can be tested without a server to host asset packs. Defaults to YES unless onDemandResourcesAssetPacksBaseURL is specified.
	iCloudContainerEnvironment
		For non-App Store exports, if the app is using CloudKit, this configures the "com.apple.developer.icloud-container-environment" entitlement. Available options: Development and Production. Defaults to Development.
	manifest : Dictionary
		For non-App Store exports, users can download your app over the web by opening your distribution manifest file in a web browser. To generate a distribution manifest, the value of this key should be a dictionary with three sub-keys: appURL, displayImageURL, fullSizeImageURL. The additional sub-key assetPackManifestURL is required when using on demand resources.
	method : String
		Describes how Xcode should export the archive. Available options: app-store, ad-hoc, package, enterprise, development, and developer-id. The list of options varies based on the type of archive. Defaults to development.
	onDemandResourcesAssetPacksBaseURL : String
		For non-App Store exports, if the app uses On Demand Resources and embedOnDemandResourcesAssetPacksInBundle isn't YES, this should be a base URL specifying where asset packs are going to be hosted. This configures the app to download asset packs from the specified URL.
	teamID : String
		The Developer Portal team to use for this export. Defaults to the team used to build the archive.
	thinning : String
		For non-App Store exports, should Xcode thin the package for one or more device variants? Available options: <none> (Xcode produces a non-thinned universal app), <thin-for-all-variants> (Xcode produces a universal app and all available thinned variants), or a model identifier for a specific device (e.g. "iPhone7,1"). Defaults to <none>.
	uploadBitcode : Bool
		For App Store exports, should the package include bitcode? Defaults to YES.
	uploadSymbols : Bool
		For App Store exports, should the package include symbols? Defaults to YES.
```

我们重点基本上用到`method`导出什么类型的包，`uploadSymbols`上传符号文件
![](http://ww1.sinaimg.cn/large/6a23c4cdgw1f9n1um53iyj20lg05wq3k.jpg)
其实导出ipa包就和xcode导出的时候能设置的参数一样，这个命令话而已。

#### 3. 注意事项，遇到的坑说一下
##### 1. 打包的时候如果要用`CODE_SIGN_IDENTITY`设置证书的话，要保证工程里面配置证书的模式不是自动的像这样，不然就无法用命令行设置成功。
![](http://ww1.sinaimg.cn/large/6a23c4cdgw1f9n1v6oqpqj211206g3yv.jpg)
脚本里面有个工具可以强制设置工程的配置，原理就是直接用`gsed`工具修改工程的`project.pbxproj`文件

```
gsed -i 's/ProvisioningStyle = Automatic;/ProvisioningStyle = Manual;/g' ${app_project_path}
gsed -i '/DevelopmentTeam = XXXXXXXXXX;/d' ${app_project_path}
```

`gsed`安装用`brew`
安装brew

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
安装gsed

```
brew install gnu-sed
```

### iOS修改ipa包签名
#### 1. 解压ipa安装包

解压为zip之后双击解压zip包就能得到文件夹Payload，里面有olinone.app

```
cp olinone.ipa olinone.zip
```

#### 2. 替换证书配置文件（文件名必须为embedded，不得自定义）

要使用的配置文件修改名字为embedded

```
cp embedded.mobileprovision Payload/olinone.app
```

#### 3. 重签名

`olinone.app.xcent`自己构造一个这个文件，就是一个plist文件改后缀名，里面`8A7TVN33SY`是苹果team名字，`com.olinone.china`是bundle id

![](http://ww1.sinaimg.cn/large/6a23c4cdgw1f9n1wtduvzj20ex052wf2.jpg)


```
certifierName="证书的SHA1值"
/usr/bin/codesign --force --sign $certifierName --entitlements olinone.app.xcent --timestamp=none Payload/olinone.app
```
#### 4. 打包

```
zip -r olinone.ipa Payload
```
