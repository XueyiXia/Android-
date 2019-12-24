# Android-

很久没有回到自己的github 了，每天都是赶项目赶项目，扎心了这两年，在开发中的一点点，我想都可以记录下来，现在记录Android 自动打包并且上传到fir 的小功能


在项目的build 目录下，记得是app 的build ，不是root 的build啧，新建一个方法，用来全局调用，等下会解释


/**
 * 自动打包方法
 * @param defaultConfig
 */
void upLoadFun(def defaultConfig,def type){
    def apkFile
    def apkDir
    rootProject.subprojects {
        if (it.name == 'app') { //此处根据grad
            // le project的名字查找到对应module下需要上传的apk
            if(type=="assembleDebug"){
                apkDir = new File(it.buildDir.path + "/outputs/apk", "debug")
            }else {
                apkDir = new File(it.buildDir.path + "/outputs/apk", "release")
            }
            println("查询APK 文件路径："+apkDir)
            def modified = 0
            def lastModifiedDir = apkDir
            //寻找当前路径下最新apk包所在路径
            apkDir.listFiles().each { dir ->
                def last = dir.lastModified()
                if (dir.isDirectory() && last > modified) {
                    modified = last
                    lastModifiedDir = dir
                }
            }

            //寻找当前路径下后缀为apk、名称包含release字样的文件
            def apkfiles = lastModifiedDir.listFiles(new FilenameFilter() {
                @Override
                boolean accept(File dir, String name) {
                    if(type=="assembleDebug"){
                        return (name.endsWith('.apk') && name.contains('debug'))
                    }else {
                        return (name.endsWith('.apk') && name.contains('release'))
                    }
                }
            })
            if (apkfiles == null || apkfiles.length == 0) {
                println("出错啦！没有找到apk文件")
                return
            }
            //由于我们的项目对每次打包的包名根据时间戳进行命名, 默认第一个为最新包,所以取first
            apkFile = apkfiles.first()

        }
    }
    println ("即将上传 $apkFile 到fir")

    //上传url
    def upLoadUrl = "http://api.fir.im/apps"
    //app 名称
    def appName =""
    appName= new String( appName.getBytes("utf-8") , "utf-8")
    //app 版本名
    def versionName =defaultConfig.versionName
    //fir api_token
    def FIR_API_TOKEN = ""	//这是自己fir 的token
    //packageName fir上应用的包名
    def packageName = defaultConfig.getApplicationId()
    //app icon 图标
    def iconPath=""	//logo 这个还没时间研究

    def buildVersion=defaultConfig.versionCode

    //获取fir上传凭证的各个字段
    def appInfo = ("curl -X POST -d type=android&bundle_id=$packageName&api_token=$FIR_API_TOKEN $upLoadUrl").execute().text
    //json解析对象拿到的是Map, 集合对应的是array, 按照这个规则取出我们需要的数据
    def appInfoBean = new JsonSlurper().parseText(appInfo)
    def token = appInfoBean["cert"]["binary"]["token"]
    def key = appInfoBean["cert"]["binary"]["key"]
    def url = appInfoBean["cert"]["binary"]["upload_url"]
    def logoUrl= appInfoBean["cert"]["icon"]["upload_url"]

    //获取到最近上传文件的时间
    def updateInfoText = ("curl -X GET -d api_token=$FIR_API_TOKEN $upLoadUrl").execute().text
    def updateInfoBean = new JsonSlurper().parseText(updateInfoText)
    println("updateInfoBean--->>" +updateInfoBean)
    // 根据最近提交到fir的时间起的log
//    def logs = ("git log --pretty=format:\"%s\" --after ").execute().text.replace(" ", "_").split("\n")
//
//    def stringBuilder = new StringBuilder()
//    if (logs.size() > 0 && logs[0] != "") {
//        int i = 0
//        for (def log : logs) {
//            stringBuilder.append(i).append(":").append(log).append(";")
//            i++
//        }
//    }
    def stringBuilder = new StringBuilder()
    stringBuilder.append(type)
    stringBuilder.append("\n")
    stringBuilder.append(getBuildTime())
    println "stringBuilder--->> :" + stringBuilder

    ("curl -X POST  --form  file=@$apkFile" +
            "  -F token=$token" +
            " -F key=$key" +
            " -F x:name=$appName" +
            " -F x:version=$versionName" +
            " -F x:build=$buildVersion" +
            " -F x:changelog=$stringBuilder" +
            " $url").execute().text


    println "上传 $apkFile 完成"

    //执行上传logo名称
//    File iconFile=new File()
    ("curl -X POST --form  file=@$apkFile" +
            "  -F token=$token" +
            " -F key=$key"+
            "$logoUrl").execute().text


}

