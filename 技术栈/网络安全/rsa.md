#### OpenSSL生成公钥私钥

- 生成 RSA 私钥和私钥:
```js
openssl genrsa -out ymt_wx_pri_2048.pem 2048
openssl rsa -in ymt_wx_pri_2048.pem -pubout -out ymt_wx_pub_2048.pem

openssl genrsa -out ymt_wx_pri_dev_2048.pem 2048
openssl rsa -in ymt_wx_pri_dev_2048.pem -pubout -out ymt_wx_pub_dev_2048.pem

openssl genrsa -out ymt_wx_pri_qa_2048.pem 2048
openssl rsa -in ymt_wx_pri_qa_2048.pem -pubout -out ymt_wx_pub_qa_2048.pem



// ali
openssl genrsa -out ymt_ali_pri_2048.pem 2048
openssl rsa -in ymt_ali_pri_2048.pem -pubout -out ymt_ali_pub_2048.pem

openssl genrsa -out ymt_ali_pri_dev_2048.pem 2048
openssl rsa -in ymt_ali_pri_dev_2048.pem -pubout -out ymt_ali_pub_dev_2048.pem

openssl genrsa -out ymt_ali_pri_qa_2048.pem 2048
openssl rsa -in ymt_ali_pri_qa_2048.pem -pubout -out ymt_ali_pub_qa_2048.pem



// lianlian
openssl genrsa -out ymt_ll_pri_2048.pem 2048
openssl rsa -in ymt_ll_pri_2048.pem -pubout -out ymt_ll_pub_2048.pem

openssl genrsa -out ymt_ll_pri_dev_2048.pem 2048
openssl rsa -in ymt_ll_pri_dev_2048.pem -pubout -out ymt_ll_pub_dev_2048.pem

openssl genrsa -out ymt_ll_pri_qa_2048.pem 2048
openssl rsa -in ymt_ll_pri_qa_2048.pem -pubout -out ymt_ll_pub_qa_2048.pem



// manual
openssl genrsa -out ymt_manual_pri_2048.pem 2048
openssl rsa -in ymt_manual_pri_2048.pem -pubout -out ymt_manual_pub_2048.pem

openssl genrsa -out ymt_manual_pri_dev_2048.pem 2048
openssl rsa -in ymt_manual_pri_dev_2048.pem -pubout -out ymt_manual_pub_dev_2048.pem

openssl genrsa -out ymt_manual_pri_qa_2048.pem 2048
openssl rsa -in ymt_manual_pri_qa_2048.pem -pubout -out ymt_manual_pub_qa_2048.pem


openssl genrsa -out ymt_zf_pri_2048.pem 2048
```

- 将传统格式的私钥转换成 PKCS#8 格式的(仅java需要?，其他语言为PKCS#1)，然后再生成公钥

`openssl pkcs8 -topk8 -inform PEM -in zf_pri_key.pem -outform PEM -nocrypt`



## tls密钥生成
```js
openssl genrsa -out ymt_ali_pri_2048.key 2048
```
