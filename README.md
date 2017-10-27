改进方案1：客户端用JS直接签名，然后上传到OSS 

demo讲述：https://bbs.aliyun.com/read/262307.html?spm=5176.bbsl211.0.0.8gmdkz
测试样例：http://oss-demo.aliyuncs.com/oss-h5-upload-js-direct/index.html   


java js改进

　/  uploader初始化代码
var uploader = new plupload.Uploader({
    runtimes: 'html5,flash,silverlight,html4',
    browse_button: 'selectfiles',
    container: document.getElementById('container'),

    url: 'http://oss.aliyuncs.com',

    init: {
        PostInit: function () {
                document.getElementById('ossfile').innerHTML = '';
                document.getElementById('postfiles').onclick = function () {
                    uploader.start();
                    return false;
                };
            },

            FilesAdded: function (up, files) {
                plupload.each(files, function (file) {
                    document.getElementById('ossfile').innerHTML += '<div id="' + file.id + '">' + file.name + ' (' + plupload.formatSize(file.size) + ')<b></b>' + '<div class="progress"><div class="progress-bar" style="width: 0%"></div></div>' + '</div>';
                });
            },

            UploadProgress: function (up, file) {
                //进度条
                var d = document.getElementById(file.id);
                d.getElementsByTagName('b')[0].innerHTML = '<span>' + file.percent + "%</span>";

                var prog = d.getElementsByTagName('div')[0];
                var progBar = prog.getElementsByTagName('div')[0]
                progBar.style.width = 2 * file.percent + 'px';
                progBar.setAttribute('aria-valuenow', file.percent);
            },

            FileUploaded: function (up, file, info) {
                console.log('uploaded')
                console.log(info.status)
                    //默认成功返回２０４
                if (info.status >= 200 || info.status < 200) {
                    document.getElementById(file.id).getElementsByTagName('b')[0].innerHTML = 'success';
                } else {
                    document.getElementById(file.id).getElementsByTagName('b')[0].innerHTML = info.response;
                }
            },

            Error: function (up, err) {
                document.getElementById('console').appendChild(document.createTextNode("\nError xml:" + err.response));
            }
    }
});

uploader.init();

//获取签名的ajax和自定义参数　OSS
$.ajax({
    type: 'get',
    dataType: 'json',
    cache: false,
    contentType: false,
    url: 'getAuthIdentity',
    success: function (obj) {
            //自定义参数
            uploader.setOption({
                multipart_params: {
                                        'key': fileKey,
                                        'policy': obj.policy,
                                        'OSSAccessKeyId': obj.accessid,
                                        'signature': obj.signature,
                                    },
                url: obj.host
            });
            uploader.start(); //调用实例对象的start()方法开始上传文件，当然你也可以在其他地方调用该方法
        },
        error: function (XmlHttpRequest, textStatus, errorThrown) {
            console.log(errorThrown)
        }
})




=================================
java 后台


/**
	 * 获取oss/s3身份验证
	 */
@RequestMapping(value = "getAuthIdentity", method = RequestMethod.GET)
@ResponseBody 
public Map < String,String > getOssIdentity(HttpServletRequest request, HttpServletResponse response, String cdn) throws Exception {

    long expireTime = 30;
			long expireEndTime = System.currentTimeMillis() + expireTime * 1000;
			String host = "http://" + "你的bucket" + ".oss-cn-qingdao.aliyuncs.com" ;
			PolicyConditions policyConds = new PolicyConditions();
			policyConds.addConditionItem(PolicyConditions.COND_CONTENT_LENGTH_RANGE, 0, 1048576000);
			com.aliyun.oss.OSSClient client = new com.aliyun.oss.OSSClient("oss-cn-qingdao.aliyuncs.com",
					"你的OSS_ACCESS_ID", "你的OSS_ACCESS_KEY");// 初始化OSS
			Date expiration = new Date(expireEndTime);
			String postPolicy = client.generatePostPolicy(expiration, policyConds);
			byte[] binaryData = postPolicy.getBytes("utf-8");
			String encodedPolicy = BinaryUtil.toBase64String(binaryData);
			String postSignature = client.calculatePostSignature(postPolicy);
			Map<String, String> respMap = identityMap("你的OSS_ACCESS_ID", host, encodedPolicy, postSignature);
			return respMap;
}

private Map < String,String > identityMap(String accessid, String host, String encodedPolicy, String postSignature) {
    Map < String,String > respMap = new LinkedHashMap < String,String > ();
    respMap.put("accessid", accessid);
    respMap.put("policy", encodedPolicy);
    respMap.put("signature", postSignature);
    respMap.put("host", host);
    return respMap;
}
