Date对象构造函数多参数的使用方法

Date对象有三个比较常用的构造方法,分别是：
```javascript
new Date()
new Date(milliseconds)
new Date(datestring)
```
以上这些构造方法基本能够满足日常的开发工作，但是如果要编写日期类的控件的话下面的方法能够提供更强大的帮助:
```javascript
new Date(year, month[, date[, hours[, minutes[, seconds[, milliseconds]]]]])
```
以下是对这7个参数的描述：
* year - 年份
* month - 月份, 计数从0开始，0代表1月，11代表12月
* date - 日期,代表当月的第几天
* hours - 小时，代表当天的第几个小时，取值范围：整数
* minutes - 分钟，代表指定hours的第几分钟，取值范围：整数
* seconds - 秒，代表指定minutes的第几秒，取值范围：整数
* milliseconds - 毫秒，代表指定seconds的第几毫秒，取值范围：整数

```javascript
new Date(2017, 2) == new Date('2017-03-01 00:00:00') // Wed Mar 01 2017 00:00:00 GMT+0800 (CST) 2017年3月1日
new Date(2017, 2, 1) === new Date('2017-03-01 00:00:00') // Wed Mar 01 2017 00:00:00 GMT+0800 (CST) 2017年3月1日
new Date(2017, 2, 2) === new Date('2017-03-02 00:00:00') // Wed Mar 02 2017 00:00:00 GMT+0800 (CST) 2017年3月2日
new Date(2017, 2, 2, 9) === new Date('2017-03-02 09:00:00') // Wed Mar 02 2017 09:00:00 GMT+0800 (CST) 2017年3月2日09点
new Date(2017, 2, 2, 9, 8) === new Date('2017-03-02 09:08:00') // Wed Mar 02 2017 09:08:00 GMT+0800 (CST) 2017年3月2日09点08分
...
```
ok，我们都知道1个小时只有60分钟，如果minutes取值大于等于60或者小于0怎办？这种情况下Date构造方法会自动进行转换，例如minute=61，则hours加1；如果小于0，则hours减1，其他入参如month，year以此类推
```javascript
new Date(2017, 2, 2, 9, 68) // Thu Mar 02 2017 10:08:00 GMT+0800 (CST)
new Date(2017, 2, 2, 9, 128) // Thu Mar 02 2017 11:08:00 GMT+0800 (CST)
new Date(2017, 2, 2, 9, -1) // Thu Mar 02 2017 08:59:00 GMT+0800 (CST)
new Date(2017, 2, 2, 9, -61) // Thu Mar 02 2017 07:59:00 GMT+0800 (CST)
...
```
好了，知道了基本的用法，接下来我们用这个构造方法计算指定月份的每一天的日期吧
```javascript
const daysOfMonth = [];
const fullYear = new Date().getFullYear();
const month = new Date().getMonth() + 1;
const lastDayOfMonth = new Date(fullYear, month, 0).getDate();
for (var i = 1; i <= lastDayOfMonth; i++) {
  console.log(fullYear + '-' + month + '-' + i);
};
// 输出
// "2017-3-1" "2017-3-2" "2017-3-3" "2017-3-4" "2017-3-5" "2017-3-6" "2017-3-7" "2017-3-8" "2017-3-9" "2017-3-10" "2017-3-11" "2017-3-12" "2017-3-13" "2017-3-14" "2017-3-15" "2017-3-16" "2017-3-17" "2017-3-18" "2017-3-19" "2017-3-20" "2017-3-21" "2017-3-22" "2017-3-23" "2017-3-24" "2017-3-25" "2017-3-26" "2017-3-27" "2017-3-28" "2017-3-29" "2017-3-30" "2017-3-31"
```
