[TOC]

## 添加背景图片

- 一般的情况下我们往往是通过backgroundImage属性来设置背景图片，但是在有的View中没有backgroundImage这个属性，这时候我们可以通过backgroundColor这个属性来添加背景图

  ```swift
  reusableViews.backgroundColor = UIColor(patternImage: UIImage(named:"bg")!)
  ```

  但是这种方法如果图片大小不够则会根据背景页面的大小进行平铺，无法将小图片进行拉伸。

- 使用以下的方法可以对图片进行拉伸：

  - 在layer层改变contents

    ```swift
    reusableViews.layer.contents = UIImage(named: "bg")?.CGImage
    ```

  - 对图片进行重绘

    ```swift
    let image = UIImage(named: "bg")
    UIGraphicsBeginImageContextWithOptions(self.view.frame.size, false, 0.1);
    image?.drawInRect(reusableViews.bounds)
    let newImage = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext();
    reusableViews.backgroundColor = UIColor(patternImage: newImage)
    ```

    对于这种方法来说如果只是简单的小部分使用还好，对内存没有什么影响，但如果要大量的去使用的话就需要考虑到对内存的消耗了。