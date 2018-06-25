# wechatminiApp_img2base64

## Aka 微信小程序从手机相册获取临时图片后转为 Base64 （奇技淫巧）

搜了各种方法都没有找到相对好实现的，然后就决定自己写一个了。

微信小程序可以使用 ```wx.chooseImage()``` 从手机相册或者相机中获取图片的临时文件，然后可以使用 ```wx.uploadFile()``` 的方式直接上传。但是假如我们要把图片转成 Base64 的形式再上传要怎么做呢？微信告诉你对不起我们不直接提供这种做法，但是我们有一个叫做 ```wx.arrayBufferToBase64()``` 的东西你们看着用哈。

坑爹呢这是！(╯‵□′)╯︵┻━┻ 你也没有把图片转成 ```arrayBuffer``` 的办法啊！好吧可以有，就是有点麻烦。

所有的代码都放在 index.js 中。

这个做法的中心思想就是要先用 ```wx.request()``` 把图片以 arrayBuffer 的格式 GET 下来，然后就可以用那个 ```wx.arrayBufferToBase64()``` 了。听起来很美好吧，我之前搜到过这种做法，在 PC 上测试是可行的，放到手机上就是找不到文件了。为什么呢？```wx.request()``` 的那个 url 需要是 http 打头的。在 PC 上测试的时候使用 ```wx.chooseImage()``` 获得的图片临时地址就是 http://tmp_random_symbols.jpg 这种格式。但是在手机上就变成了 wxfile://tmp/....jpg 之类的了，就没法 GET 。

坑爹呢这是！(╯‵□′)╯︵┻━┻ 冷静冷静，没有 http 我们就给它一个 http。（直接把 wxfile 之流的替换成 http 是不行的哦亲测无效。）

那怎么办呢？

首先我们要用知道一个叫做“临时素材”的概念，你可以把图片作为临时素材在微信服务器中保存三天。能放到服务器上我们不就有 http 了吗！（兴奋地搓手手）

说干就干。

首先（咦）要使用微信的临时服务器，就要先获取一个 access_token (https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140183)
具体做法就是

```
wx.request({
  url: 'https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=yourAPPID&secret=yourAPPSECRET',
  method: 'GET',
  success: function(token_res) {
    var token = token_res.data.access_token
  }
})
```

这里的 yourAPPID 和 yourAPPSECRET 你可以在自己的管理界面里获得。经过这个步骤你就有了一个 access_token。

然后就可以上传临时文件了。因为是微信的临时服务器嘛我们直接用 ```wx.uploadFile()``` 就好啦。

```
wx.uploadFile({
  url: "https://api.weixin.qq.com/cgi-bin/media/upload?access_token=ACCESS_TOKEN&type=image",
  filePath: yourdata.tempFilePaths[0],
  name: 'media',
  success: function(upload_res) {
    var media_id = JSON.parse(upload_res.data).media_id
  }
})
```

media_id 就是你上传的文件的一个 key，等下要根据这个 media_id 获得你刚刚传上去的临时文件。需要注意的一点是使用 ```wx.uplodaFile()``` 上传成功上传文件之后的返回的那个 upload_res.data 里的信息是一个 JSON 的字符串而不是 JSON ，所以要先把它 parse 成 JSON 才能获取到 media_id 的信息。

再然后就可以用 ```wx.request()``` 的方式把这个图以 arrayBuffer 的形式 GET 了。

```
wx.request({
  url: "https://api.weixin.qq.com/cgi-bin/media/get?access_token=ACCESS_TOKEN&media_id=MEDIA_ID",
  method: 'GET',
  header: {
    'content-type': 'image/jpeg'
  },
  responseType: 'arraybuffer',
  success: function(resp_data) {
    var base64 = wx.arrayBufferToBase64(resp_data.data)
  }
}）
```

这里的 ```resp_data.data``` 就是转成了 arrayBuffer 的图片。```base64``` 就是我们的临时图片转成的 Base64 了。

把这几个 request 一嵌套你就可以为所欲为啦……（逃）
