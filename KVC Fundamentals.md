# 键值编码机制

这篇文章描述了KVC（键值编码）的基本机制。

##### 官方文档

[Key-Value Coding Fundamentals](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/BasicPrinciples.html)

##### 键和键的路径

键是一个描述对象具体属性的字符。原则上，一个键与一个寄存器方法或者随着接收值而变化的实例的名称相符。这些键必须使用ASCII编码，以小写字母开头，同时不包含空格。

以下是一些可以使用的键：`payee`, `openingBalance`, `transactions` 以及 `amount`.

键的路径是一个用来遍历指定对象的属性序列，这个序列是以点分隔的字符串。这个键里的第一个属性是与接收者相关的，同时随后的每一个键都相对前一个属性的值相关。

例如，键的路径 `address.street` 可以从接收者对象得到 `address` 属性当中的值，然后可以查到与 `address` 对象相关联的 `street` 属性。

##### 利用KVC机制获取属性的值

`valueForKey` 方法返回与接收者相关的特定键值。如果相对这个键没有变化的寄存器和实例，那么接收者会给自己发动一条 `valueForUndefinedKey:` 的消息。默认的 `valueForUndefinedKey` 的实现会抛出一个[`NSUndefinedKeyException`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Protocols/NSKeyValueCoding_Protocol/index.html#//apple_ref/doc/c_ref/NSUndefinedKeyException) ，它的子类可以重载这个行为。

相似地， `valueForKeyPath:` 方法也返回与接收者相关的特定键值。任何在键路径上的非键值编码的键只要符合条件，其对象就会获取一个 `valueForUndefinedKey` 的信息。

`dictionaryWithValuesForKeys:` 方法可以获取一个与接收者相关的键值数组。返回的 `NSDictionary` 包含所有在数组当中的键的值。

>阅读笔记：一些泛型类，例如 `NSArray`, `NSSet`, 以及 `NSDictionary` 是不能包含 `nil` 作为其中的值的。其实，你可以用一个 `NSNull` 来代替 `nil` 。 `NSNull` 可以作为一个单一的实例出现在类属性里用来代替 `nil` 。 `dictionaryWithValuesForKeys:` 和 `setValuesForKeysWithDictionary:` 这两个方法的实现可以自动地把 `null` 转化成 `NSNull` ，所以你也不必要在类里明确地测试 `NSNull` 的转化了。

当一个值是返回给一个包含了多对多关系的键，而且这个键并不是路径中的最后一个键时，这个返回的值就会是包含了所有在多对多的键右侧的键的所有值的集合。例如，获取路径 `transactions.payee` 的值会返回一个包括了所有交易中所有收款人对象的数组。这个规则对多维数组也适用，路径 `accounts.transactions.payee` 将会返回所有账户的所有交易的所有收款人对象的数组。（译者注：路径中的单词均表达了字面意思。account:账户，transaction:交易，payee:收款人）

##### 利用KVC设置属性的值

`setValue:forKey:` 方法可以给指定的与接收者相关的键赋值为给定的数值。 `setValue:forKey:` 方法的实现可以解开 `NSValue` 的对象，声明纯数据和结构并分配给属性。参考文档 [Scalar and Structure Support](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/DataTypes.html#//apple_ref/doc/uid/20002171-BAJEAIEE) 可以深入了解关于包装和解开包装的语法和用法。

如果指定的键并不存在，接收者就会被发送 `setValue:forUndefinedKey:` 的消息。默认的 `setValue:forUndefinedKey:` 实现会抛出一个 [`NSUndefinedKeyException`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Protocols/NSKeyValueCoding_Protocol/index.html#//apple_ref/doc/c_ref/NSUndefinedKeyException)。同时，子类可以重载这个方法，用默认的方法处理请求。

`setValue:forKeyPath:` 和上面的方法做法相似，但是它允许处理一个键的路径甚至是单独一个键。

最后， `setValuesForKeysWithDictionary:` 可以把给定的字典赋值给接收者的属性，利用字典的键来辨别属性。默认的实现对每个键值对引用 `setValue:forKey:` ，然后按要求用 `NSNull` 代替 `nil` 。

你还需要额外考虑一件事，在你给一个没有对象的属性赋值为 `nil` 会发生什么呢？这种情况下，接收者会给自己发送 `setNilValueForKey:` 的消息。默认的 `setNilValueForKey:` 实现会抛出一个 [`NSInvalidArgumentException`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Miscellaneous/Foundation_Constants/index.html#//apple_ref/doc/c_ref/NSInvalidArgumentException)。你的应用可以重载这个方法来替代默认值或者标记的值，然后用新的值调用 `setValue:forKey:` 。

##### KVC & 点的语法

Objective-C中的点语法和KVC是相互垂直的两种用法。无论是否使用点句法，你都可以使用KVC，也无论你是否使用KVC，你都可以使用点句法。虽然点句法随时都可以使用，但是用法有不同。（译者注：其实就是指Objective-C原生支持点句法，不过官方文档写得真是太有文采了，实在做不到原意翻译）在KVC里，点句法是用来界定键值当中的元素的。一定记住，当你用点句法调用一个属性的时候，你调用的是接收者的基本寄存器方法。

你可以使用键值的方法调用一个属性，下面给出了一个定义类的样例：

```
@interface MyClass
@property NSString *stringProperty;
@property NSInteger integerProperty;
@property MyClass *linkedInstance;
@end
```

你可以通过KVC在一个实例中调用它的属性：

```
MyClass *myInstance = [[MyClass alloc] init];
NSString *string = [myInstance valueForKey:@"stringProperty"];
[myInstance setValue:@2 forKey:@"integerProperty"];
```

为了分辨点句法在KVC和原生语法之间的区别，你可以参考以下代码：

```
MyClass *anotherInstance = [[MyClass alloc] init];
myInstance.linkedInstance = anotherInstance;
myInstance.linkedInstance.integerProperty = 2;
```

以下代码和上例等价：

```
MyClass *anotherInstance = [[MyClass alloc] init];
myInstance.linkedInstance = anotherInstance;
[myInstance setValue:@2 forKeyPath:@"linkedInstance.integerProperty"];
```
