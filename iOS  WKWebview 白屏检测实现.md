# iOS  WKWebview 白屏检测实现



## 前言

​	自ios8推出wkwebview以来，极大改善了网页加载速度及内存泄漏问题，逐渐全面取代笨重的UIWebview。尽管高性能、高刷新的WKWebview在混合开发中大放异彩表现优异，但加载网页过程中出现异常白屏的现象却仍然屡见不鲜，且现有的api协议处理捕捉不到这种异常case，造成用户无用等待体验很差。

​	针对业务场景需求，实现加载白屏检测。考虑采用字节跳动团队提出的webview优化技术方案。在合适的加载时机对当前webview可视区域截图，并对此快照进行像素点遍历，如果非白屏颜色的像素点超过一定的阈值，认定其为非白屏，反之重新加载请求。



## 获取快照

​	ios官方提供了简易的获取webview快照接口，通过异步回调拿到当前可视区域的屏幕截图。

```
- (void)takeSnapshotWithConfiguration:(nullable WKSnapshotConfiguration *)snapshotConfiguration completionHandler:(void (^)(UIImage * _Nullable snapshotImage, NSError * _Nullable error))completionHandler API_AVAILABLE(ios(11.0));
```

​	其中snapshotConfiguration 参数可用于配置快照大小范围，默认截取当前客户端整个屏幕区域。由于可能出现导航栏成功加载而内容页却空白的特殊情况，导致非白屏像素点数增加对最终判定结果造成影响，考虑将其剔除。

```
- (void)judgeLoadingStatus:(WKWebView *)webview {
    if (@available(iOS 11.0, *)) {
        if (webView && [webView isKindOfClass:[WKWebView class]]) {
            
            CGFloat statusBarHeight =  [[UIApplication sharedApplication] statusBarFrame].size.height; //状态栏高度
            CGFloat navigationHeight =  webView.viewController.navigationController.navigationBar.frame.size.height; //导航栏高度
            WKSnapshotConfiguration *shotConfiguration = [[WKSnapshotConfiguration alloc] init];
            shotConfiguration.rect = CGRectMake(0, statusBarHeight + navigationHeight, _webView.bounds.size.width, (_webView.bounds.size.height - navigationHeight - statusBarHeight)); //仅截图检测导航栏以下部分内容
            [_webView takeSnapshotWithConfiguration:shotConfiguration completionHandler:^(UIImage * _Nullable snapshotImage, NSError * _Nullable error) {
                //todo
            }];
        }
    }
}
```



## 缩放快照

​	为了提升检测性能，考虑将快照缩放至1/5，减少像素点总数，从而加快遍历速度。

```
- (UIImage *)scaleImage: (UIImage *)image {
    CGFloat scale = 0.2;
    CGSize newsize;
    newsize.width = floor(image.size.width * scale);
    newsize.height = floor(image.size.height * scale);
    if (@available(iOS 10.0, *)) {
        UIGraphicsImageRenderer * renderer = [[UIGraphicsImageRenderer alloc] initWithSize:newsize];
          return [renderer imageWithActions:^(UIGraphicsImageRendererContext * _Nonnull rendererContext) {
                        [image drawInRect:CGRectMake(0, 0, newsize.width, newsize.height)];
                 }];
    }else{
        return image;
    }
}
```

缩小前后性能对比（实验环境：iPhone11同一页面下）：

- 缩放前白屏检测：

![img](src/beforeadd.jpg)

![img](src/beforeadd2.jpg)

​		耗时20ms

- 缩放后白屏检测：

![img](src/afteradd.jpg)

![img](src/afteradd2.jpg)

​		耗时13ms

​	注意这里有个**小坑**。由于缩略图的尺寸在 原图宽高*缩放系数后可能不是整数，在布置画布重绘时默认向上取整，这就造成画布比实际缩略图大（混蛋啊 摔！）。在遍历缩略图像素时，会将图外画布上的像素纳入考虑范围，导致实际白屏页 像素占比并非100% 如图所示。因此使用floor将其尺寸大小**向下取整**。



## 遍历快照

  遍历快照缩略图像素点，对白色像素(R:255 G: 255 B: 255)占比大于95%的页面，认定其为白屏。

```
- (BOOL)searchEveryPixel:(UIImage *)image {
    CGImageRef cgImage = [image CGImage];
    size_t width = CGImageGetWidth(cgImage);
    size_t height = CGImageGetHeight(cgImage);
    size_t bytesPerRow = CGImageGetBytesPerRow(cgImage); //每个像素点包含r g b a 四个字节
    size_t bitsPerPixel = CGImageGetBitsPerPixel(cgImage);
    
    CGDataProviderRef dataProvider = CGImageGetDataProvider(cgImage);
    CFDataRef data = CGDataProviderCopyData(dataProvider);
    UInt8 * buffer = (UInt8*)CFDataGetBytePtr(data);
    
    int whiteCount = 0;
    int totalCount = 0;
    
    for (int j = 0; j < height; j ++ ) {
        for (int i = 0; i < width; i ++) {
            UInt8 * pt = buffer + j * bytesPerRow + i * (bitsPerPixel / 8);
            UInt8 red   = * pt;
            UInt8 green = *(pt + 1);
            UInt8 blue  = *(pt + 2);
//            UInt8 alpha = *(pt + 3);
        
            totalCount ++;
            if (red == 255 && green == 255 && blue == 255) {
                whiteCount ++;
            }
        }
    }
    float proportion = (float)whiteCount / totalCount ;
    NSLog(@"当前像素点数：%d,白色像素点数:%d , 占比: %f",totalCount , whiteCount , proportion );
    if (proportion > 0.95) {
        return YES;
    }else{
        return NO;
    }
}
```



## 总结

```
typedef NS_ENUM(NSUInteger,webviewLoadingStatus) {
    
    WebViewNormalStatus = 0, //正常
    
    WebViewErrorStatus, //白屏
    
    WebViewPendStatus, //待决
};


// 判断是否白屏
- (void)judgeLoadingStatus:(WKWebview *)webview  withBlock:(void (^)(webviewLoadingStatus status))completionBlock{
    webviewLoadingStatus __block status = WebViewPendStatus;
    if (@available(iOS 11.0, *)) {
        if (webview && [webview isKindOfClass:[WKWebView class]]) {
            
            CGFloat statusBarHeight =  [[UIApplication sharedApplication] statusBarFrame].size.height; //状态栏高度
            CGFloat navigationHeight = webview.viewController.navigationController.navigationBar.frame.size.height; //导航栏高度
            WKSnapshotConfiguration *shotConfiguration = [[WKSnapshotConfiguration alloc] init];
            shotConfiguration.rect = CGRectMake(0, statusBarHeight + navigationHeight, webview.bounds.size.width, (webview.bounds.size.height - navigationHeight - statusBarHeight)); //仅截图检测导航栏以下部分内容
            [webview takeSnapshotWithConfiguration:shotConfiguration completionHandler:^(UIImage * _Nullable snapshotImage, NSError * _Nullable error) {
                if (snapshotImage) {
                    CGImageRef imageRef = snapshotImage.CGImage;
                    UIImage * scaleImage = [self scaleImage:snapshotImage];
                    BOOL isWhiteScreen = [self searchEveryPixel:scaleImage];
                    if (isWhiteScreen) {
                       status = WebViewErrorStatus;
                    }else{
                       status = WebViewNormalStatus;
                    }
                }
                if (completionBlock) {
                    completionBlock(status);
                }
            }];
        }
    }
}

// 遍历像素点 白色像素占比大于95%认定为白屏
- (BOOL)searchEveryPixel:(UIImage *)image {
    CGImageRef cgImage = [image CGImage];
    size_t width = CGImageGetWidth(cgImage);
    size_t height = CGImageGetHeight(cgImage);
    size_t bytesPerRow = CGImageGetBytesPerRow(cgImage); //每个像素点包含r g b a 四个字节
    size_t bitsPerPixel = CGImageGetBitsPerPixel(cgImage);
    
    CGDataProviderRef dataProvider = CGImageGetDataProvider(cgImage);
    CFDataRef data = CGDataProviderCopyData(dataProvider);
    UInt8 * buffer = (UInt8*)CFDataGetBytePtr(data);
    
    int whiteCount = 0;
    int totalCount = 0;
    
    for (int j = 0; j < height; j ++ ) {
        for (int i = 0; i < width; i ++) {
            UInt8 * pt = buffer + j * bytesPerRow + i * (bitsPerPixel / 8);
            UInt8 red   = * pt;
            UInt8 green = *(pt + 1);
            UInt8 blue  = *(pt + 2);
//            UInt8 alpha = *(pt + 3);
        
            totalCount ++;
            if (red == 255 && green == 255 && blue == 255) {
                whiteCount ++;
            }
        }
    }
    float proportion = (float)whiteCount / totalCount ;
    NSLog(@"当前像素点数：%d,白色像素点数:%d , 占比: %f",totalCount , whiteCount , proportion );
    if (proportion > 0.95) {
        return YES;
    }else{
        return NO;
    }
}

//缩放图片
- (UIImage *)scaleImage: (UIImage *)image {
    CGFloat scale = 0.2;
    CGSize newsize;
    newsize.width = floor(image.size.width * scale);
    newsize.height = floor(image.size.height * scale);
    if (@available(iOS 10.0, *)) {
        UIGraphicsImageRenderer * renderer = [[UIGraphicsImageRenderer alloc] initWithSize:newsize];
          return [renderer imageWithActions:^(UIGraphicsImageRendererContext * _Nonnull rendererContext) {
                        [image drawInRect:CGRectMake(0, 0, newsize.width, newsize.height)];
                 }];
    }else{
        return image;
    }
}
```



​	仅需在合适的view生命周期内回调使用该函数方法即可检测出页面状态是否白屏，且性能损耗可忽略不计。



## 声明

本文为作者原创，转载请注明出处。

本文原发表地址链接🔗 [掘金](https://juejin.im/post/6885298718174609415)

