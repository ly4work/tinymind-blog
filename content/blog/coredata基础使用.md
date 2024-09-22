---
title: CoreData基础使用
date: 2024-09-22T06:26:00.580Z
---

# CoreData基础使用: 创建Entity+增删改查

这篇文章主要是总结一些常见的 CoreData 的操作。

我们先定义数据结构如下，这里我们的数据结构对应代码的生成方式是 Xcode 自动生成的，看图右侧的 Class → Codegen 部分。

![coredata](https://fanthus.github.io/assets/img/coredata_2.35b009b4.png)

> 对于每个属性还有更细致的设置，比如如果你不想要某个属性持久化，而是临时使用，可以设置属性为 Transient，等这部分详细的说明就不多说了可以参考官方文档 [Configuring Attributes(opens new window)](https://developer.apple.com/documentation/coredata/modeling_data/configuring_attributes)

有的时候创建完 Entity 之后，在 Swift 文件中输入对应的 Entity 类名，还是会报错说找不到这个新建的 Entity。可以试试下面几个办法，参考[这里的回答(opens new window)](https://stackoverflow.com/questions/58651871/xcode-11-doesnt-recognize-core-data-entity)

1. 重新 build 失败之后，关掉 Xcode 重新启动，这时候 Xcode 可能就会识别到你新建的这个 Entity。（Xcode 垃圾）
2. 选中当前的 Entity，去 Editor -> Create NSManagedObject Subclass 新建一个实例。其实理论上你是不用创建的，因为 Xcode 会自动生成的，但是创建这个文件的操作可能会触发 Xcode 识别的过程，哪怕之后删了也能正常 Build..
3. 检查一下你新建的 Entity 的 Attribute 到底有没有问题，比如 AttributeType 是不是正常的。

> 有的时候新增一些实例(Entity)属性，Xcode 不认的时候也可以尝试以上办法。

## [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E5%88%9B%E5%BB%BA%E5%AE%9E%E4%BE%8B)创建实例

创建 Student 实例，有几种方法

① 使用 `NSManagedObject` 的 designated initializer 来进行初始化，但是需要先构建 `NSEntityDescription`，总体流程比较繁琐。`NSEntityDescription` 是 `Entity` 的描述类，

```
let managedObjectModel:NSManagedObjectModel? = context.persistentStoreCoordinator?.managedObjectModel
let entity:NSEntityDescription? = managedObjectModel?.entitiesByName["Student"]
let student1:Student = NSManagedObject.init(entity: entity!, insertInto: context) as! Student
student1.name = "One"
```

② 使用 `NSEntityDescription` 的类方法来构建 `Student` 实例，它是第一种方法的改进版本，这个可以参考官方文档的说明，和第一种构建方法没啥区别，就是方便调用者。

```
let student2 = NSEntityDescription.insertNewObject(forEntityName: "Student", into: context) as!  Student
student2.name = "Two"
```

③ 使用 `NSManagedObject` 的便捷(convenience)初始化方法来初始化 `Student` 实例。

```
let stu = Student(context: context)
stu.name = "Three
```

推荐第三种，是最简单最直接的构建对应 Entity 实例的方法，不用进行任何类型转换。

# [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E6%8F%92%E5%85%A5%E6%95%B0%E6%8D%AE)插入数据

创建 `NSManagedObject` 实例并不代表这个数据被持久化了，我们需要手动调用 `NSManagedObjectContext` 的 save 方法去进行持久化。

我之前有个错误的习惯是，习惯在创建好 `NSManagedObject` 实例后，再调用 `NSManagedObjectContext` 的 insert 方法插入一下，然后再 save。其实没有必要。因为在创建 `NSManagedObject` 实例的时候我们已经传入了 context 的参数，就表示已经在对应的 context里注册了。我理解 insert 的使用场景是，如果你创建 `NSManagedObject` 的时候没有传 context，或者你需要再另外一个 context 里也保存当前 Entity 实例，那就需要重新 insert一下，否则没有必要。

## [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E5%8D%95%E4%B8%AA%E6%95%B0%E6%8D%AE%E6%8F%92%E5%85%A5)单个数据插入

直接上代码如下，没什么新鲜的东西，值得注意的是，最好不要直接 try! 或者 try? 去直接 save，而是要捕获异常，这样当出问题的时候我们就不知道为什么出问题，线上的 App 最好弹出错误的提示，这样我们就能有针对性的帮助用户排查问题。

```
let context:NSManagedObjectContext = ((UIApplication.shared.delegate as? AppDelegate)?.persistentContainer.viewContext)!
let stu = Student(context: context)
stu.name = "Fanthus"
do {
    try context.save()
} catch {
    print("save entity error \(error)")
}
```

## [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E6%89%B9%E9%87%8F%E6%95%B0%E6%8D%AE%E6%8F%92%E5%85%A5)批量数据插入

使用 BatchRequest，一般这种场景是大量的数据才会用，比如从服务端请求回来大量的数据一次写回本地。

```
let context:NSManagedObjectContext = ((UIApplication.shared.delegate as? AppDelegate)?.persistentContainer.viewContext)!
let batchInserRequest = createBatchInsertRequest()
do {
    let insertResult = try context.execute(batchInserRequest) as! NSBatchInsertResult
    print(insertResult)
} catch {
    print("batch insert failed \(error)")
}

var students: [Student] {
    var students = [Student]()
    let stu1 = Student.init(context: context)
    stu1.name = "Batch1"
    let stu2 = Student.init(context: context)
    stu2.name = "Batch2"
    students.append(stu1)
    students.append(stu2)
    return students
}
//创建批量插入请求实例
func createBatchInsertRequest() -> NSBatchInsertRequest {
    var index = 0
    let total = students.count
    let batchInserRequest = NSBatchInsertRequest(entity: Student.entity(), managedObjectHandler: { obj in
        guard index < total else { return true }
        if let stuObj = obj as? Student {
            let student = self.students[index]
            stuObj.name = student.name
        }
        index += 1
        return false
    })
    return batchInserRequest
}
```

# [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E6%9F%A5%E8%AF%A2%E6%95%B0%E6%8D%AE)查询数据

数据被本地化后，我们使用 `NSFetchRequest` 去获取当前已经存在的数据，举个例子，如果我们获取本地某个 Entity 的全部数据的话，代码如下

```
let context:NSManagedObjectContext = ((UIApplication.shared.delegate as? AppDelegate)?.persistentContainer.viewContext)!
//两种构建 FetchRequest 的方法，不过第二种方式有别的用途
//推荐
let studentFetch1: NSFetchRequest<Student> = Student.fetchRequest()
//不推荐
let studentFetch2 = NSFetchRequest<NSFetchRequestResult>(entityName: "Student")
do {
    let messages = try context.fetch(studentFetch)
    //容易出错的获取结果的方式，是通过 execute 方式获取
		//① 报错:Cannot fetch without an NSManagedObjectContext in scope，这个 API 需要在 ViewContext 下使用
		//let students = try studentFetch.execute()
		//② 拿到的是 NSAsynchronousFetchResult 实例，没有办法继续获取
		//let fetchResult = try context.execute(studentFetch)
    print(students)
} catch {
    fatalError("Failed to fetch employees: \(error)")
}
```

💡 官方文档里的 API 用的是第二种方式，现在 API 的实现似乎已经变了。

但是我们经常是要根据一些条件来进行查询的，如何设置查询条件呢？可以直接给 FetchRequest 设置 Predicate，比如下面代码检索 name 中包含 batch 关键字的条目。

```
let studentFetch: NSFetchRequest<Student> = Student.fetchRequest()
studentFetch.predicate = NSPredicate.init(format: "name CONTAINS[c] %@", "batch")
do {
    let students = try context.fetch(studentFetch)
    print(students)
} catch {
    fatalError("Failed to fetch employees: \(error)")
}
```

关于 Predicate 的使用，可以参考官方的 [Predicate Programming Guide (opens new window)](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/AdditionalChapters/Introduction.html#//apple_ref/doc/uid/TP40001789).

# [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E5%88%A0%E9%99%A4%E6%95%B0%E6%8D%AE)删除数据

删除数据之前需要你先找出待删除的数据，所以需要结合上面的查询数据的功能来进行删除。

## [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E5%8D%95%E4%B8%AA%E5%88%A0%E9%99%A4)单个删除

删除单个数据的代码如下，一定要记得在要调用 save 方法，因为 context.delete 方法只是标记了对象应该被删除，当真正提交更改的时候才会被删除。而 save 就是进行提交更改的方法（commit unsaved changed methods）。

```
do {
    let context:NSManagedObjectContext = ((UIApplication.shared.delegate as? AppDelegate)?.persistentContainer.viewContext)!
    if let students = fetchStudents() {
        students.forEach { student in
            context.delete(student)
        }
        try context.save()
    }
} catch  {
    print("delete error \(error)")
}
```

CoreData 删除其实有两种情况，一种就是数据还没有被持久化，一种是数据已经被持久化。未被持久化的情况直接 delete 的话，就是直接从 context 中移除了，调不调 save 没啥区别。后一种就是上面说的这种情况。

## [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E6%89%B9%E9%87%8F%E5%88%A0%E9%99%A4)批量删除

批量删除需要先构建批量删除请求，然后执行请求。

```
//构建批量删除请求
let context:NSManagedObjectContext = ((UIApplication.shared.delegate as? AppDelegate)?.persistentContainer.viewContext)!
let fetch = NSFetchRequest<NSFetchRequestResult>(entityName: "Student") //这里构建 FetchRequest 的方式是通过 Entity 名字构建的
fetch.predicate = NSPredicate.init(format: "name CONTAINS[c] %@", "batch")
let request = NSBatchDeleteRequest(fetchRequest: fetch)
//执行批量删除请求
do {
    let result = try context.execute(request)
    print("result \(result)")
} catch {
    fatalError("Failed to execute request: \(error)")
}
```

# [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E4%BF%AE%E6%94%B9%E6%95%B0%E6%8D%AE)修改数据

修改数据需要将待修改的数据查找出来，然后再进行修改。

## [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E5%8D%95%E4%B8%AA%E4%BF%AE%E6%94%B9)单个修改

单个修改并没有特殊的 API，就是查找数据出来之后，直接在 `NSManagedObject` 上面对属性进行修改就好了。

```
do {
    let context:NSManagedObjectContext = ((UIApplication.shared.delegate as? AppDelegate)?.persistentContainer.viewContext)!
    if let students = try fetchStudents(), students.count > 0 {
        let student = students.first
        student?.name = "hello world"
        try context.save()
    }
} catch  {
    print("save error \(error)")
}
```

## [#](https://fanthus.github.io/2022/11/24/coredata%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-2/#%E6%89%B9%E9%87%8F%E4%BF%AE%E6%94%B9)批量修改

和批量删除一样，也是先构建批量更新请求的实例，然后再执行这个批量更新请求。

```
do {
    let context:NSManagedObjectContext = ((UIApplication.shared.delegate as? AppDelegate)?.persistentContainer.viewContext)!
		//创建 BatchUpdate	实例
    let request = NSBatchUpdateRequest(entityName: "Student")
    request.predicate = NSPredicate.init(format: "name CONTAINS[c] %@", "batch")
    request.propertiesToUpdate = ["name":"new_stu"] //提交待更新数据以及值
		//执行批量更新
    try context.execute(request)
} catch {
    print("update failed \(error)")
}
```

其实可以看到不管是修改数据还是删除数据都是还是需要通过 predicate 查找出来待修改或者待删除的条目，所以查找数据这个知识点是掌握修改数据和删除数据的前提。

参考地址:

1. [CoreData-Configuring Entities(opens new window)](https://developer.apple.com/documentation/coredata/modeling_data/configuring_entities)
2. [CoreData-Generating Code(opens new window)](https://developer.apple.com/documentation/coredata/modeling_data/generating_code)
3. [CoreData-Creating a Managed Object Model(opens new window)](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreData/KeyConcepts.html)
4. [WWDC20 10017 - Core Data 杂项与准则(opens new window)](https://kukushi.github.io/blog/core-data-Sundries-and-maxims)
5. [Modern, Efficient Core Data (opens new window)](https://www.kodeco.com/14958063-modern-efficient-core-data)#批量插入数据改进 CoreData 性能
6. [Core Data Batch Programming Guide(opens new window)](https://www.notion.so/CoreData-CloudKit-ccb9e515679e4bb290d4a0d2b3602f77)

---
