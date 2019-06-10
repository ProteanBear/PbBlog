# iOS:设置Grouped风格的UITableView的顶部位置
iOS开发时使用UITableViewController的Grouped风格时，表格视图顶部总有一个怪怪的空白地区很奇怪（如下图），那么有没有办法把它去掉呢？

<center><img src="https://res.cloudinary.com/proteanbear/image/upload/v1559718281/blog/tips/grouped_blank.jpg" alt="带空白的Grouped风格UITableView" data-src="https://res.cloudinary.com/proteanbear/image/upload/v1559718281/blog/tips/grouped_blank.jpg" data-size="160x285" /></center>

其实也简单，这个空白产生的原因其实是iOS7以后导航栏有模糊效果后，iOS的SDK中自动对UITableView的顶部边距进行了自动设置，不清楚为什么Grouped风格的顶部边距就设置的多了╮(╯_╰)╭！

基于这种原理只要取消掉这个自动设置就好了，iOS也提供了相应的方法，在对应的UITableViewController的viewDidLoad中加入下面代码：

```swift
override func viewDidLoad(){
	super.viewDidLoad()

	//插入如下代码
	self.automaticallyAdjustsScrollViewInsets=false
}	
```

加上后，顶部位置明显是从顶部状态条开始了：

<center><img src="https://res.cloudinary.com/proteanbear/image/upload/v1559718281/blog/tips/grouped_not_set_section.jpg" alt="修改后被遮挡住的UITableView" data-src="https://res.cloudinary.com/proteanbear/image/upload/v1559718281/blog/tips/grouped_not_set_section.jpg" data-size="160x285" /></center>

只要再根据自己的表格header高度设置下第一个section的高度就好了：

```swift
override func tableView(tableView: UITableView,
	heightForHeaderInSection section: Int) -> CGFloat
{
    return section==0 ? (64+34):34
}
```

<center><img src="https://res.cloudinary.com/proteanbear/image/upload/v1559718281/blog/tips/grouped_right.jpg" alt="最终结果" data-src="https://res.cloudinary.com/proteanbear/image/upload/v1559718281/blog/tips/grouped_right.jpg" data-size="160x285" /></center>

