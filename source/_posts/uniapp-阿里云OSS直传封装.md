---
title: uniApp代码-阿里云OSS直传封装
toc: true
date: 2022-08-02 00:00:00
tags: [uniApp, 代码片段]
categories:
  - [前端技术, 代码片段]
---
本文使用的方式是直接上传，不需要向后端获取凭据。优点是上传方便，缺点是本地存储Key较为不安全。

### 一、目录结构

```text
-- common
   -- crypto
      -- base64.js
			-- crypto.js
			-- hmac.js
			-- sha1.js
	 -- oss
			-- index.js
			-- OSSConfig.js
```

### 二、使用例

```html
<template>
  <view style="display: flex;flex-direction: column;">
    <button @click="uploadImage">点我上传图片</button>
    <button @click="uploadVideo">点我上传视频</button>
    <image v-if="imageKey" :src="ossPath+imageKey" mode="aspectFit"></image>
    <video v-if="videoKey" :src="ossPath+videoKey"></video>
  </view>
</template>

<script>
  const app = getApp();
  import OSS from '@/common/oss/index.js'
  export default {
    data() {
      return {
        ossPath: app.globalData.ossPath,
        imageKey: '',
        videoKey: ''
      }
    },
    methods: {
      uploadImage() {
        const _this = this;
        OSS.ossUploadImage({
          success(res) {
            _this.imageKey = res;
          }
        })
      },
      uploadVideo() {
        const _this = this;
        OSS.ossUploadVideo({
          success(res) {
            _this.videoKey = res;
          }
        })
      }
    }
  }
</script>
```

### 三、代码

* oss/OSSConfig.js

  ```javascript
  import Crypto from '@/common/crypto/crypto.js';
  import '@/common/crypto/hmac.js';
  import '@/common/crypto/sha1.js';
  import { Base64 } from '@/common/crypto/base64.js';
  
  let date = new Date()
  date = date.setHours(date.getHours() + 1)
  let extime = "" + new Date(date).toISOString()
  let policyText = {
    "expiration": extime,
    "conditions": [
      ["content-length-range", 0, 1024 * 1024 * 100] // 设置上传文件的大小限制 
    ]
  };
  let config = {
    accessid: '这里填写OSS的AccessId',
    accesskey: '这里填写OSS的AccessKey',
    osshost: getApp().globalData.ossPath,
    policyBase64: Base64.encode(JSON.stringify(policyText))
  }
  let message = config.policyBase64;
  let bytes = Crypto.HMAC(Crypto.SHA1, message, config.accesskey, {
    asBytes: true
  });
  let signature = Crypto.util.bytesToBase64(bytes);
  let timetamp = new Date().getTime();
  let OSSConfig = {
    name: 'aliyun',
    host: config.osshost,
    accessid: config.accessid,
    signature: signature,
    policyBase64: config.policyBase64,
  }
  export default OSSConfig;
  ```

* oss/index.js

  ```javascript
  /**
   * 阿里云OSS工具类
   * @author myd
   * @date 2022-07-18 11:51:49
   */
  import OSSConfig from "./OSSConfig"
  
  /**
   * 生成一个随机的Key
   */
  function storeKey() {
    let s = [];
    let hexDigits = "0123456789abcdef";
    for (let i = 0; i < 36; i++) {
      s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1);
    }
    s[14] = "4";
    s[19] = hexDigits.substr((s[19] & 0x3) | 0x8, 1);
    s[8] = s[13] = s[18] = s[23] = "-";
    return s.join("");
  }
  
  /**
   * 根据当天日期在OSS端生成文件夹
   */
  function storeFolder() {
    const date = new Date();
    const formatNumber = n => {
      n = n.toString()
      return n[1] ? n : '0' + n
    }
    return [date.getFullYear(), date.getMonth() + 1, date.getDate()].map(formatNumber).join('-')
  }
  
  /**
   * 阿里云OSS上传文件, 所有具体功能的工具函数均基于此
   * 注意, resolve时一定为上传成功, 返回OSS上的Key
   * @param filePath 待上传文件的URI
   * @param key 存储桶中的目标文件名
   * @param folder 存储桶中的目标文件夹
   */
  export function ossUpload(filePath, key = storeKey(), folder = storeFolder()) {
    return new Promise((resolve, reject) => {
      if (folder && folder?.length > 0) {
        if (folder[0] == "/") folder = folder.slice(1, folder.length)
        if (folder[folder.length - 1] != "/") folder += "/"
        key = folder + key
      }
      const filePrefixArr = filePath.split(".")
      key += `.${filePrefixArr[filePrefixArr.length - 1]}`
      let config = {
        url: OSSConfig.host,
        name: 'file',
        filePath,
        formData: {
          key,
          policy: OSSConfig.policyBase64,
          OSSAccessKeyId: OSSConfig.accessid,
          success_action_status: '200',
          signature: OSSConfig.signature,
        },
        success(res) {
          if (res.errMsg.includes("uploadFile:ok")) {
            resolve(key)
          } else {
            reject(res)
          }
        },
        fail(err) {
          reject(err)
        }
      }
      uni.uploadFile(config)
    })
  }
  
  /**
   * 阿里云OSS上传图片
   * @param {compressed, key, folder, success, fail} compressed: 是否压缩 key: 存储桶中的目标文件名 folder: 存储桶中的目标文件夹
   */
  export function ossUploadImage({
    key,
    folder,
    compressed = true, //是否压缩
    success, //成功时的回调
    fail //失败时的回调
  }) {
    const sizeType = [compressed ? 'compressed' : 'original']
    uni.chooseImage({
      count: 1,
      sizeType,
      success(res) {
        console.log(res)
        ossUpload(res.tempFilePaths[0], key, folder).then(success).catch(fail)
      },
      fail
    })
  }
  
  /**
   * 阿里云OSS上传视频
   * @param { key, folder, sourceType, compressed, maxDuration, camera, success, fail} 
   * key: 存储桶中的目标文件名 folder: 存储桶中的目标文件夹 其它参数同uni.chooseVideo(mpWeixin) 
   */
  export function ossUploadVideo({
    key,
    folder,
    sourceType = ['album', 'camera'], //album 从相册选视频, camera 使用相机拍摄
    compressed = true, //是否压缩所选的视频源文件
    maxDuration = 60, //拍摄视频最长拍摄时间, 单位秒。最长支持 60 秒
    camera = 'back', //调用相机方向, 'front'、'back', 默认'back'
    success, //成功时的回调
    fail //失败时的回调
  }) {
    uni.chooseVideo({
      sourceType,
      compressed,
      maxDuration,
      camera,
      success(res) {
        ossUpload(res.tempFilePath, key, folder).then(success).catch(fail)
      },
      fail
    })
  }
  const OSS = {
    ossUploadVideo,
    ossUploadImage,
    ossUpload
  }
  export default OSS;
  ```

* crypto/base64.js

  ```javascript
  export const Base64 = {
    _keyStr: "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=",
    encode: function(input) {
      var output = "";
      var chr1, chr2, chr3, enc1, enc2, enc3, enc4;
      var i = 0;
      input = Base64._utf8_encode(input);
      while (i < input.length) {
        chr1 = input.charCodeAt(i++);
        chr2 = input.charCodeAt(i++);
        chr3 = input.charCodeAt(i++);
        enc1 = chr1 >> 2;
        enc2 = ((chr1 & 3) << 4) | (chr2 >> 4);
        enc3 = ((chr2 & 15) << 2) | (chr3 >> 6);
        enc4 = chr3 & 63;
        if (isNaN(chr2)) {
          enc3 = enc4 = 64;
        } else if (isNaN(chr3)) {
          enc4 = 64;
        }
        output = output +
          this._keyStr.charAt(enc1) + this._keyStr.charAt(enc2) +
          this._keyStr.charAt(enc3) + this._keyStr.charAt(enc4);
      }
      return output;
    },
  
    decode: function(input) {
      var output = "";
      var chr1, chr2, chr3;
      var enc1, enc2, enc3, enc4;
      var i = 0;
      input = input.replace(/[^A-Za-z0-9\+\/\=]/g, "");
      while (i < input.length) {
        enc1 = this._keyStr.indexOf(input.charAt(i++));
        enc2 = this._keyStr.indexOf(input.charAt(i++));
        enc3 = this._keyStr.indexOf(input.charAt(i++));
        enc4 = this._keyStr.indexOf(input.charAt(i++));
        chr1 = (enc1 << 2) | (enc2 >> 4);
        chr2 = ((enc2 & 15) << 4) | (enc3 >> 2);
        chr3 = ((enc3 & 3) << 6) | enc4;
        output = output + String.fromCharCode(chr1);
        if (enc3 != 64) {
          output = output + String.fromCharCode(chr2);
        }
        if (enc4 != 64) {
          output = output + String.fromCharCode(chr3);
        }
      }
      output = Base64._utf8_decode(output);
      return output;
    },
  
    _utf8_encode: function(string) {
      string = string.replace(/\r\n/g, "\n");
      var utftext = "";
      for (var n = 0; n < string.length; n++) {
        var c = string.charCodeAt(n);
        if (c < 128) {
          utftext += String.fromCharCode(c);
        } else if ((c > 127) && (c < 2048)) {
          utftext += String.fromCharCode((c >> 6) | 192);
          utftext += String.fromCharCode((c & 63) | 128);
        } else {
          utftext += String.fromCharCode((c >> 12) | 224);
          utftext += String.fromCharCode(((c >> 6) & 63) | 128);
          utftext += String.fromCharCode((c & 63) | 128);
        }
      }
      return utftext;
    },
  
    _utf8_decode: function(utftext) {
      var string = "";
      var i = 0;
      var c = c1 = c2 = 0;
      while (i < utftext.length) {
        c = utftext.charCodeAt(i);
        if (c < 128) {
          string += String.fromCharCode(c);
          i++;
        } else if ((c > 191) && (c < 224)) {
          c2 = utftext.charCodeAt(i + 1);
          string += String.fromCharCode(((c & 31) << 6) | (c2 & 63));
          i += 2;
        } else {
          c2 = utftext.charCodeAt(i + 1);
          c3 = utftext.charCodeAt(i + 2);
          string += String.fromCharCode(((c & 15) << 12) | ((c2 & 63) << 6) | (c3 & 63));
          i += 3;
        }
      }
      return string;
    }
  }
  ```

* crypto/crypto.js

  ```javascript
  var base64map = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
  let Crypto = {};
  var util = Crypto.util = {
    rotl: function(n, b) {
      return (n << b) | (n >>> (32 - b));
    },
    rotr: function(n, b) {
      return (n << (32 - b)) | (n >>> b);
    },
    endian: function(n) {
      if (n.constructor == Number) {
        return util.rotl(n, 8) & 0x00FF00FF |
          util.rotl(n, 24) & 0xFF00FF00;
      }
      for (var i = 0; i < n.length; i++)
        n[i] = util.endian(n[i]);
      return n;
    },
    randomBytes: function(n) {
      for (var bytes = []; n > 0; n--)
        bytes.push(Math.floor(Math.random() * 256));
      return bytes;
    },
    stringToBytes: function(str) {
      var bytes = [];
      for (var i = 0; i < str.length; i++)
        bytes.push(str.charCodeAt(i));
      return bytes;
    },
    bytesToString: function(bytes) {
      var str = [];
      for (var i = 0; i < bytes.length; i++)
        str.push(String.fromCharCode(bytes[i]));
      return str.join("");
    },
    stringToWords: function(str) {
      var words = [];
      for (var c = 0, b = 0; c < str.length; c++, b += 8)
        words[b >>> 5] |= str.charCodeAt(c) << (24 - b % 32);
      return words;
    },
    bytesToWords: function(bytes) {
      var words = [];
      for (var i = 0, b = 0; i < bytes.length; i++, b += 8)
        words[b >>> 5] |= bytes[i] << (24 - b % 32);
      return words;
    },
    wordsToBytes: function(words) {
      var bytes = [];
      for (var b = 0; b < words.length * 32; b += 8)
        bytes.push((words[b >>> 5] >>> (24 - b % 32)) & 0xFF);
      return bytes;
    },
    bytesToHex: function(bytes) {
      var hex = [];
      for (var i = 0; i < bytes.length; i++) {
        hex.push((bytes[i] >>> 4).toString(16));
        hex.push((bytes[i] & 0xF).toString(16));
      }
      return hex.join("");
    },
    hexToBytes: function(hex) {
      var bytes = [];
      for (var c = 0; c < hex.length; c += 2)
        bytes.push(parseInt(hex.substr(c, 2), 16));
      return bytes;
    },
    bytesToBase64: function(bytes) {
      if (typeof btoa == "function") return btoa(util.bytesToString(bytes));
      var base64 = [],
        overflow;
      for (var i = 0; i < bytes.length; i++) {
        switch (i % 3) {
          case 0:
            base64.push(base64map.charAt(bytes[i] >>> 2));
            overflow = (bytes[i] & 0x3) << 4;
            break;
          case 1:
            base64.push(base64map.charAt(overflow | (bytes[i] >>> 4)));
            overflow = (bytes[i] & 0xF) << 2;
            break;
          case 2:
            base64.push(base64map.charAt(overflow | (bytes[i] >>> 6)));
            base64.push(base64map.charAt(bytes[i] & 0x3F));
            overflow = -1;
        }
      }
      if (overflow != undefined && overflow != -1)
        base64.push(base64map.charAt(overflow));
      while (base64.length % 4 != 0) base64.push("=");
      return base64.join("");
    },
    base64ToBytes: function(base64) {
      if (typeof atob == "function") return util.stringToBytes(atob(base64));
      base64 = base64.replace(/[^A-Z0-9+\/]/ig, "");
      var bytes = [];
      for (var i = 0; i < base64.length; i++) {
        switch (i % 4) {
          case 1:
            bytes.push((base64map.indexOf(base64.charAt(i - 1)) << 2) |
              (base64map.indexOf(base64.charAt(i)) >>> 4));
            break;
          case 2:
            bytes.push(((base64map.indexOf(base64.charAt(i - 1)) & 0xF) << 4) |
              (base64map.indexOf(base64.charAt(i)) >>> 2));
            break;
          case 3:
            bytes.push(((base64map.indexOf(base64.charAt(i - 1)) & 0x3) << 6) |
              (base64map.indexOf(base64.charAt(i))));
            break;
        }
      }
      return bytes;
    }
  };
  Crypto.mode = {};
  export default Crypto;
  ```

* crypto/hmac.js

  ```javascript
  import Crypto from '@/common/crypto/crypto.js';
  (function() {
    // Shortcut
    var util = Crypto.util;
    Crypto.HMAC = function(hasher, message, key, options) {
      // Allow arbitrary length keys
      key = key.length > hasher._blocksize * 4 ?
        hasher(key, {
          asBytes: true
        }) :
        util.stringToBytes(key);
      // XOR keys with pad constants
      var okey = key,
        ikey = key.slice(0);
      for (var i = 0; i < hasher._blocksize * 4; i++) {
        okey[i] ^= 0x5C;
        ikey[i] ^= 0x36;
      }
      var hmacbytes = hasher(util.bytesToString(okey) +
        hasher(util.bytesToString(ikey) + message, {
          asString: true
        }), {
          asBytes: true
        });
      return options && options.asBytes ? hmacbytes :
        options && options.asString ? util.bytesToString(hmacbytes) :
        util.bytesToHex(hmacbytes);
    };
  })();
  ```

* crypto/sha1.js

  ```javascript
  import Crypto from '@/common/crypto/crypto.js'
  (function() {
    // Shortcut
    var util = Crypto.util;
    // Public API
    var SHA1 = Crypto.SHA1 = function(message, options) {
      var digestbytes = util.wordsToBytes(SHA1._sha1(message));
      return options && options.asBytes ? digestbytes :
        options && options.asString ? util.bytesToString(digestbytes) :
        util.bytesToHex(digestbytes);
    };
    // The core
    SHA1._sha1 = function(message) {
      var m = util.stringToWords(message),
        l = message.length * 8,
        w = [],
        H0 = 1732584193,
        H1 = -271733879,
        H2 = -1732584194,
        H3 = 271733878,
        H4 = -1009589776;
      // Padding
      m[l >> 5] |= 0x80 << (24 - l % 32);
      m[((l + 64 >>> 9) << 4) + 15] = l;
      for (var i = 0; i < m.length; i += 16) {
        var a = H0,
          b = H1,
          c = H2,
          d = H3,
          e = H4;
        for (var j = 0; j < 80; j++) {
          if (j < 16) w[j] = m[i + j];
          else {
            var n = w[j - 3] ^ w[j - 8] ^ w[j - 14] ^ w[j - 16];
            w[j] = (n << 1) | (n >>> 31);
          }
          var t = ((H0 << 5) | (H0 >>> 27)) + H4 + (w[j] >>> 0) + (
            j < 20 ? (H1 & H2 | ~H1 & H3) + 1518500249 :
            j < 40 ? (H1 ^ H2 ^ H3) + 1859775393 :
            j < 60 ? (H1 & H2 | H1 & H3 | H2 & H3) - 1894007588 :
            (H1 ^ H2 ^ H3) - 899497514);
          H4 = H3;
          H3 = H2;
          H2 = (H1 << 30) | (H1 >>> 2);
          H1 = H0;
          H0 = t;
        }
        H0 += a;
        H1 += b;
        H2 += c;
        H3 += d;
        H4 += e;
      }
      return [H0, H1, H2, H3, H4];
    };
  
    // Package private blocksize
    SHA1._blocksize = 16;
  })();
  ```
