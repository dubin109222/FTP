# FTP
搭建内网FTP

1.使用轻量级的https://github.com/svenstaro/miniserve  可以使用brew 安装
2.使用sh 脚本创建证书，证书创建之后放到钥匙串同意权限

这个有很多方案都可以，Mac自带的Apache或者Nginx...
我选用了这个 https://github.com/svenstaro/miniserve，直接映射磁盘指定目录，并且轻量级。
因为同时需要两个https 、http，所以起了两个服务 （记得自行配置开机自启动）

## 端口号 证书路径 自行修改
#  nohup commend &  这种格式是为了后台运行

cd ~/
# 启动https服务
nohup miniserve -p 8000 -z -v -t "iOS开发部FTP" -U -u --tls-cert ~/Desktop/online_auto_run/server-crt.crt --tls-key ~/Desktop/online_auto_run/server-key.key  ~/Documents/ftpRoot &

# 启动http服务
nohup miniserve -p 8001 -v -t "iOS开发部FTP" -U -u  ~/Documents/ftpRoot &

这样就可以通过http://本机ip:8001/ 及 https://本机:8000/进行访问了 （也可以当做内网的一个FTP用😆）

2.管理脚本

我是基于fastlane 、 ruby实现的，代码都比较简单，最终会返回的html页面地址，内网打开即可
主要步骤：

拷贝ipa文件到服务映射的磁盘目录
生成plist文件
生成下载二维码图片
生成下载页面html

#  " --- 尝试 配置本次打包的 内网下载配置 --- "
    #入参 pj_scheme: app scheme 用来生成路径（不用中文的title是因为 路径是url的一部分）
    #入参 pj_env: ”1“/”0“ 用来生成路径
    #入参 pj_ver: 主版本号 用来生成路径的一部分 
    #入参 pj_ipa_path: 本次打包的ipa文件路径 （绝对路径）
    #入参 pj_main_bundleID: 主包名即可 用来生成plist文件
    #入参 pj_title: app名称 用来生成plist文件
    #返回值： {pageUrl:  xxxxxxxx.html }
    lane :try_moveIPA_to_serverPath do |variable|
        ipa_server_rootpath = File.expand_path("~/Documents/ftpRoot/itms-services")
        if Dir.exist?(ipa_server_rootpath) == false 
            puts "未检测到 ipa_server 目录，跳过内网下载处理~"
            next {}
        end
        puts "检测到 ipa_server 目录，自动配置当前ipa支持内网下载 ~~~~"

        pj_scheme = variable[:pj_scheme]
        pj_env = variable[:pj_env] == "1" ? "_product_" : "_test_"      
          
        env_Dir_path = File.join(ipa_server_rootpath, pj_scheme, pj_env) 
        if Dir.exist?(env_Dir_path) 
            puts "自动删除 3 天前的包...."
            Dir.entries(env_Dir_path).each do |item|
                # 去除.开头的文件
                next if File.basename(item).start_with?(".")
                # 
                item_abs_path = File.join(env_Dir_path, item)
                sh("rm", "-R", item_abs_path) if (Time.new - File.mtime(item_abs_path) > (3 * 24 * 3600))
            end
        else
            # 创建文件夹
            sh("mkdir", "-p", env_Dir_path)
        end

        pj_ver = variable[:pj_ver]

        new_pj_ver = "#{pj_ver}-" + Time.new.strftime('%Y-%m-%d_%H-%M')
        current_dir_path = File.join(env_Dir_path, new_pj_ver)
        sh("rm", "-R", current_dir_path) if Dir.exist?(current_dir_path) 
        sh("mkdir", "-p", current_dir_path)


        # 为了在安装tsl证书前能访问html， ci服务器上跑了两个服务
        http_server_domain = "http://本机IP:8001/itms-services"
        https_server_domain = "https://本机IP:8000/itms-services"
        root_CA_URL = "http://本机IP:8001/CA/ROOT_CA_CERT.crt"

        # copy过来ipa
        puts "copy ipa 文件...."
        sh("cp", "-f", File.expand_path(variable[:pj_ipa_path]), "#{current_dir_path}/ipa.ipa")
        ipa_URL = File.join(http_server_domain, pj_scheme, pj_env, new_pj_ver, "ipa.ipa")

        # 生成plist
        puts "生成mainfest.plist文件...."
        plist_base64 = "PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0iVVRGLTgiPz4KPCFET0NUWVBFIHBsaXN0IFBVQkxJQyAiLS8vQXBwbGUvL0RURCBQTElTVCAxLjAvL0VOIiAiaHR0cDovL3d3dy5hcHBsZS5jb20vRFREcy9Qcm9wZXJ0eUxpc3QtMS4wLmR0ZCI+CjxwbGlzdCB2ZXJzaW9uPSIxLjAiPgo8ZGljdD4KCTxrZXk+aXRlbXM8L2tleT4KCTxhcnJheT4KCQk8ZGljdD4KCQkJPGtleT5hc3NldHM8L2tleT4KCQkJPGFycmF5PgoJCQkJPGRpY3Q+CgkJCQkJPGtleT5raW5kPC9rZXk+CgkJCQkJPHN0cmluZz5zb2Z0d2FyZS1wYWNrYWdlPC9zdHJpbmc+CgkJCQkJPGtleT51cmw8L2tleT4KCQkJCQk8c3RyaW5nPl9hcHBfaXBhVVJMXzwvc3RyaW5nPgoJCQkJPC9kaWN0PgoJCQk8L2FycmF5PgoJCQk8a2V5Pm1ldGFkYXRhPC9rZXk+CgkJCTxkaWN0PgoJCQkJPGtleT5idW5kbGUtaWRlbnRpZmllcjwva2V5PgoJCQkJPHN0cmluZz5fYXBwX2J1bmRsZWlkXzwvc3RyaW5nPgoJCQkJPGtleT5idW5kbGUtdmVyc2lvbjwva2V5PgoJCQkJPHN0cmluZz5fYXBwX3Zlcl88L3N0cmluZz4KCQkJCTxrZXk+a2luZDwva2V5PgoJCQkJPHN0cmluZz5zb2Z0d2FyZTwvc3RyaW5nPgoJCQkJPGtleT50aXRsZTwva2V5PgoJCQkJPHN0cmluZz5fYXBwX3RpdGxlXzwvc3RyaW5nPgoJCQk8L2RpY3Q+CgkJPC9kaWN0PgoJPC9hcnJheT4KPC9kaWN0Pgo8L3BsaXN0Pgo="
        plist_Content = Base64.decode64(plist_base64)
        # 有几个占位符需要替换掉 _app_title_ 、_app_ver_ 、 _app_bundleid_ 、 _app_ipaURL_ （其实只有_app_ipaURL_是核心）
        plist_Content = plist_Content.gsub("_app_ipaURL_", ipa_URL)
        plist_Content = plist_Content.gsub("_app_ver_", pj_ver)
        plist_Content = plist_Content.gsub("_app_bundleid_", variable[:pj_main_bundleID])
        plist_Content = plist_Content.gsub("_app_title_", variable[:pj_title])
        # 写入文件
        File.open("#{current_dir_path}/manifest.plist", "w+:utf-8") do |lines|  #读写模式。如果文件存在，则重写已存在的文件。如果文件不存在，则创建一个新文件用于读写
            lines.write(plist_Content) 
        end
        plist_URL = File.join(https_server_domain, pj_scheme, pj_env, new_pj_ver, "manifest.plist")

        install_URL = "itms-services://?action=download-manifest&url=#{plist_URL}"
        

        # 创建安装二维码图片文件
        qr = RQRCode::QRCode.new(install_URL, :level=>:h)
        png = qr.as_png(
            resize_gte_to: false,
            resize_exactly_to: false,
            fill: 'white',
            color: 'black',
            size: 180,
            border_modules: 0,
            module_px_size: 0,
            file: "#{current_dir_path}/QRImg.png" # path to write
        )
        qR_Img_URL = File.join(http_server_domain, pj_scheme, pj_env, new_pj_ver, "QRImg.png")


        # 生成html
        html_base64 = "PCFET0NUWVBFIGh0bWw+CjxodG1sPgogICAgPGhlYWQ+CiAgICAgIDxtZXRhIG5hbWU9InZpZXdwb3J0ImNvbnRlbnQ9IndpZHRoPWRldmljZS13aWR0aCwgaW5pdGlhbC1zY2FsZT0xLjAsIG1pbmltdW0tc2NhbGU9MC41LCBtYXhpbXVtLXNjYWxlPTIuMCwgdXNlci1zY2FsYWJsZT15ZXMiLz4KICAgICAgPG1ldGEgaHR0cC1lcXVpdj0iQ29udGVudC1UeXBlIiBjb250ZW50PSJ0ZXh0L2h0bWw7IGNoYXJzZXQ9VVRGLTgiPgogICAgPC9oZWFkPgogICAgPGJvZHk+CiAgICAgICAgPGg0PuaJi+acuummluasoeS4i+i9veivt+WFiCLngrnlh7vlronoo4VTU0zor4HkuaYi77yM5bm25qC55o2u5o+Q56S65a6J6KOFL+S/oeS7u+ivgeS5pjwvaDQ+CiAgICAgICAgPGEgdGl0bGU9ImlQaG9uZSIgaHJlZj0iX1Jvb3RfQ0FfVVJMXyI+6aaW5qyh6ZyA54K55q2k5a6J6KOFU1NM6K+B5LmmPC9hPgogICAgICAgIOWuieijhea1geeoi+WQjCLmipPljIXor4HkuaYi77yM5a6J6KOF5ZCO6ZyA6KaB5omL5Yqo5byA5ZCv5Y+XIFNTTCDkv6Hku7vvvIjliY3lvoDigJzorr7nva7igJ0+4oCc6YCa55So4oCdPuKAnOWFs+S6juacrOacuuKAnT7igJzor4Hkuabkv6Hku7vorr7nva7igJ3vvIkKICAgICAgICA8aHI+CiAgICAgICAgPGg0PuWmguaenOS9oOaYr+aJi+acuuaJk+W8gOeahOacrOmhtemdojwvaDQ+CiAgICAgICAgPGEgaHJlZj0iX2l0bXMtc2VydmljZXNfdXJsXyIgY2xhc3M9ImFwcF9saW5rIj7ngrnlh7vlronoo4U8L2E+CiAgICAgICAgPGhyPgogICAgICAgIDxoND7lpoLmnpzkvaDmmK/nlLXohJHmiZPlvIDnmoTmnKzpobXpnaLvvIzor7fnlKjmiYvmnLrmiavov5nkuKrkuoznu7TnoIHlronoo4U8L2g0PgogICAgICAgIDxpbWcgc3JjPSJfcXJfaW1hZ2VfdXJsXyI+CiAgICA8L2JvZHk+CjwvaHRtbD4="
        html_Content = Base64.decode64(html_base64).force_encoding("UTF-8")
        # 有几个占位符需要替换掉 _itms-services_url_ 、 _qr_image_url_ 、 _Root_CA_URL_
        html_Content = html_Content.gsub("_itms-services_url_", install_URL)
        html_Content = html_Content.gsub("_qr_image_url_", qR_Img_URL)
        html_Content = html_Content.gsub("_Root_CA_URL_", root_CA_URL)
        # 写入文件
        File.open("#{current_dir_path}/index.html", "w+:utf-8") do |lines|  #读写模式。如果文件存在，则重写已存在的文件。如果文件不存在，则创建一个新文件用于读写
            lines.write(html_Content) 
        end

        # 这个html 用http访问
        {"pageUrl" => File.join(http_server_domain, pj_scheme, pj_env, new_pj_ver, "index.html")}        
    end
