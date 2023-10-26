<a name="HT8d0"></a>
### 下单
POST<br />`domain + /collection/place/create`<br />`Content-Type: application/x-www-form-urlencoded;charset=utf-8`<br />请求参数

| **参数名** | **参数类型** | **是否必填** | **示例、说明** |
| --- | --- | --- | --- |
| unique | string | Y | 用户唯一标识 |
| order_number | string | Y | 订单号 |
| fee | int | Y | 金额(单位:分)<br />当商户类型为person时需要能被100整除，即为正整数的**元 |
| jump_url | string | Y | 订单完成跳转url |
| notice_url | string | Y | 订单完成异步通知url |
| vendor_type | string | Y | 商户类型<br /> |
| product | string | N | 见下方 支付产品<br />(若`vendor_type`使用<br />`缩写方式`，则此参数忽略，否则必填) |
| desc | string | N | 商品描述 |
| sign | string | Y | 签名 |
 
<br />返回数据
```json
{
    "status": "success",
    "message": "下单成功",
    "code": 0,
    "platform_order": "2023101549545242", // 本平台订单号
    "outer_order": "abcd1234", // 用户订单号,下单时传入的
    // 支付跳转地址
    "payment_url": "http://**.***.com/collection/import/*/*/2023101549545242",
    "sign": "D0B1116E29C19F648C49903504C69884" // 签名
}
```
 
<a name="nuWZb"></a>
### 同步跳转
GET<br />用户支付完成后，平台会自动`302跳转`至下单时传入的`jump_url`中并携带以下参数

| **参数名** | **参数类型** | **是否必填** | **示例、说明** |
| --- | --- | --- | --- |
| order_num | string | Y | 用户订单号 |
| platform_order | string | Y | 平台订单号 |
| rand_str | string | Y | 随机字符串 |
| fee | int | Y | 用户实付金额(单位:分) |
| sign | string | Y | 签名 |

<a name="GTp7M"></a>
### 订单查询
POST<br />`domain + /api/order/trade/query`<br />`Content-Type: application/x-www-form-urlencoded;charset=utf-8`<br />请求参数

| **参数名** | **参数类型** | **是否必填** | **示例、说明** |
| --- | --- | --- | --- |
| unique | string | Y | 用户唯一标识 |
| order_num | string | N | 平台订单号 |
| outer_order | string | N | 用户订单号 |
| rand_str | string | Y | 随机字符串 |
| sign | string | Y | 签名 |

`**order_num** 与 **outer_order** 二选一即可`<br />返回数据
```json
{
    "status": 200,
    "message": "success",
    "data": {
        "trade_status": "FAIL", // 支付状态 FAIL 未支付 SUCCESS 已支付
        "order_num": "2023101549545242", // 平台侧订单号
        "outer_order": "2023101549545242", // 用户侧订单号
        "rand_str": "cctv", // 查询传入的随机字符串
        "sign": "B185043C79644CADD9DA2C5ED984D984" // 签名
    }
}
```
<a name="xMUbr"></a>
### 异步回调
**请求方式：**POST<br />`Content-Type: application/x-www-form-urlencoded;charset=utf-8`<br />**回调URL：**该链接是通过基础下单接口中的请求参数“notice_url”来设置的，请确保回调URL是外部可正常访问的，且不能携带后缀参数，否则可能导致商户无法接收到微信的回调通知信息。回调URL示例：“http://**.**.com/**/**”<br />**通知规则**<br />用户支付完成后，平台会把相关支付结果和用户信息发送给商户，商户需要接收处理该消息，并返回应答。<br />对后台通知交互时，如果收到商户的应答不符合规范或超时，平台认为通知失败，平台会通过一定的策略定期重新发起通知，尽可能提高通知的成功率，但不保证通知最终能成功。（通知频率为15s/15s/30s/3m/10m/20m/30m/30m/30m/60m/3h/3h/3h/6h/6h - 总计 24h4m）<br />在发送通知后接收到字符串`SUCCESS`时视为应答成功<br />**通知报文**

| **参数名** | **参数类型** | **是否必填** | **示例、说明** |
| --- | --- | --- | --- |
| order_num | string | Y | 用户侧订单号 |
| platform_order | string | Y | 平台订单号 |
| rand_str | string | Y | 下单传入的随机字符串 |
| original_price | int | Y | 原下单金额(单位:分) |
| real_price | int | Y | 实际支付金额(单位:分) |
| order_message | string | Y | 订单状态描述 |
| order_status | string | Y | 订单状态<br />CREATE 订单创建<br />SUCCESS 已支付<br />CANCEL 已取消<br />REFUND 已退款 |
| sign | string | Y | 签名 |

<a name="Ix2gq"></a>
### 签名方法

1. 将参数名按ASCLL字典序顺序排序
2. 将参数按照key=>value键值对的形式拼接，形成一个字符串 类似 { a=b&b=c&c=d }
3. 在字符串末尾拼接{ &cert=`cert` } cert 为每个用户的 密钥
4. 将最终生成的字符串进行md5加密 得到一个32位字符串
5. 将md5值转为大写
<a name="zk8mG"></a>
#### php示例代码
```php
/**
* 签名
* @param array $arr  参与签名的数组
* @param string $cert 用户密钥
* @return string
*/
protected static function makeSign(array $arr, string $cert): string
{
    $arr = array_filter($arr, function ($val) {
      	return ($val === '' || $val === null) ? false : true;
    });
    if (isset($arr['sign'])) {
      	unset($arr['sign']);
    }
    ksort($arr);
    $str = http_build_query($arr);
    $str = urldecode($str) . '&cert=' . $cert;
    $str = md5($str);
    return  strtoupper($str);
}
```
<a name="cN5a2"></a>
### 错误码
| **code** | **message** |
| --- | --- |
| 0 | 成功 |
| 2000 | 下单失败 |

<a name="uLuYV"></a>
### 状态码
| **status** | **message** |
| --- | --- |
| success | 成功 |
| 其他 | 失败 |
