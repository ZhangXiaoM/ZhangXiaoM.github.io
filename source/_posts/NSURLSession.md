---
title: NSURLSession
date: 2015-08-21 17:19:55
tags: iOS API 详解
---

本文翻译自 https://www.raywenderlich.com/110458/nsurlsession-tutorial-getting-started   

原作者：[Ken Toh](https://www.raywenderlich.com/u/kentoh)    

译者：X.M.Zhang       

----
当一个app从服务端获取用户数据更新社交媒体信息和下载远程的文件到磁盘的时候，就会用到移动应用的核心技术：HTTP网络请求。为了帮助开发者处理大量的网络请求，Apple提供了`NSURLSession`，这是一套通过HTTP请求来完成上传和下载的完整的API。  

在本教程中，你将学会怎样去用`NSURLSession`来创建一个Half Tunes应用，这个应用的作用是接入[iTunes Search API](https://www.apple.com/itunes/affiliates/resources/documentation/itunes-store-web-service-search-api.html)，搜索API提供的30s的预览音乐，并且下载你选择的音乐。完成后的app将提供后台传输，让用户暂停、继续、取消正在下载的音乐等功能。   

## 入门

[点击此处下载项目](http://www.raywenderlich.com/wp-content/uploads/2016/01/HalfTunes-Starter.zip)  

项目包含搜索歌曲、播放歌曲的用户界面，和一些解析JSON、播放路径的帮助方法，可以让你专心的去实现app的网络部分。  

运行你下载的项目，将会看到如下的界面： 

![](https://cdn2.raywenderlich.com/wp-content/uploads/2015/08/Simulator-Screen-Shot-12-Aug-2015-11.10.57-pm-281x500.png) 

在搜索框输入一个想搜索的内容，点击搜索按钮。仍然是空的界面，不要方，你将通过`NSURLSession`的调用来完善他的功能。  

## NSURLSession概述

在开始之前，让我们先了解一下`NSURLSession`和他的组成类。   

`NSURLSession`既使用一个类，又使用一整套类去完成基于HTTP/HTTPS的请求   

![](https://cdn1.raywenderlich.com/wp-content/uploads/2015/08/Screen-Shot-2015-08-20-at-12.27.21-am.png)  

`NSURLSession`是响应发送和接收HTTP请求的关键对象，使用`NSURLSessionConfiguration`来创建它。三种配置：    

* `defaultSessionConfiguration`:创建一个默认配置的session对象，可以访问永久的全部的磁盘缓存、证书和保存的cookie对象。    
* `ephemeralSessionConfiguration`:除了只能在内存中保存和这个session有关的数据之外，其他的和默认配置相同，可以理解为“私有的”session。    
* `backgroundSessionConfiguration`:用这个配置创建的session对象，可以在后台进行下载和上传操作，即使app暂停或者终止。 

你也可以使用`NSURLSessionConfiguration`配置session的其他属性，比如：超时、缓存策略、附加的HTTP头部等。[所有的配置选项](https://developer.apple.com/library/mac/documentation/Foundation/Reference/NSURLSessionConfiguration_class/index.html#//apple_ref/occ/cl/NSURLSessionConfiguration)  

`NSURLSessionTask`是一个抽象类，session用来生成task去做实际的工作，比如：获取数据、下载和上传文件等。   

Apple提供了三种task：  

* `NSURLSessionDataTask`:通过HTTP GET方法，从服务端获取数据。
* `NSURLSessionUploadTask`:顾名思义，用来向web服务器上传文件，一般使用HTTP POST和PUT方法。
* `NSURLSessionDownloadTask`:从远程服务器下载文件到一个临时的本地路径。

![](http://www.raywenderlich.com/wp-content/uploads/2015/08/Screen-Shot-2015-08-20-at-12.27.27-am.png)   

你可以随时暂停、继续、取消任务。`NSURLSessionDownloadTask`支持断点续传。  

一般来说，`NSURLSession`可以通过两种方式（完成block和代理）进行任务完成成功或者报错的回调。

现在，你已经了解了`NSURLSession`的主要功能，可以进行理论与实践相结合了。  

## 查询跟踪    

你将在之前下载的项目内添加一些代码，当用户搜索一些内容的时候，查询iTunes Search API。  

在`SearchViewController.swift`类的顶部添加下面的代码：

```swift
// 1
let defaultSession = NSURLSession(configuration: NSURLSessionConfiguration.defaultSessionConfiguration())
// 2
var dataTask: NSURLSessionDataTask?
```

这两行代码的作用是：

&emsp;1.创建了`NSURLSession`，并且使用默认的配置初始化。   
&emsp;2.声明了一个`NSURLSessionDataTask`变量，将用它来向iTunes Search web服务器进行HTTP GET请求实现用户的搜索。当用户每次重新输入内容搜索的时候，这个task都会被重新初始化并且利用。    

现在，用下面的代码替换`searchBarSearchButtonClicked(_:) `这个方法：

```swift
func searchBarSearchButtonClicked(searchBar: UISearchBar) {
  dismissKeyboard()
 
  if !searchBar.text!.isEmpty {
    // 1
    if dataTask != nil {
      dataTask?.cancel()
    }
    // 2
    UIApplication.sharedApplication().networkActivityIndicatorVisible = true
    // 3
    let expectedCharSet = NSCharacterSet.URLQueryAllowedCharacterSet()
    let searchTerm = searchBar.text!.stringByAddingPercentEncodingWithAllowedCharacters(expectedCharSet)!
    // 4
    let url = NSURL(string: "https://itunes.apple.com/search?media=music&entity=song&term=\(searchTerm)")
    // 5
    dataTask = defaultSession.dataTaskWithURL(url!) {
      data, response, error in
      // 6
      dispatch_async(dispatch_get_main_queue()) {
        UIApplication.sharedApplication().networkActivityIndicatorVisible = false
      }
      // 7
      if let error = error {
        print(error.localizedDescription)
      } else if let httpResponse = response as? NSHTTPURLResponse {
        if httpResponse.statusCode == 200 {
          self.updateSearchResults(data)
        }
      }
    }
    // 8
    dataTask?.resume()
  }
}
```
按照注释数字的顺序看：     

&emsp;1.每当用户点击搜索的时候，检查data task是不是已经初始化，如果是，取消这个task，因为我们在重用同一个task，取消的task是上次搜索的task。    
&emsp;2.开启状态栏的网络活动指示，告诉用户已经在搜索。    
&emsp;3.在把用户的输入字符串作为参数进行请求之前，对字符串调用`stringByAddingPercentEncodingWithAllowedCharacters(_:) `方法，确保是正确的字符格式。    
&emsp;4.使用转化格式后的字符拼接字符串构造了一个`NSURL`，作为请求的url。   
&emsp;5.通过之前创建的`defaultSession`实例化一个`NSURLSessionDataTask`，去执行HTTP GET请求。这个方法传入了之前构造的`url`和一个完成回调block。    
&emsp;6.在task的完成回调中，回到主线程隐藏网络活动知识器。    
&emsp;7.如果请求成功，调用`updateSearchResults(_:)`将响应的NSData数据解析成`track`(model)。    
&emsp;8.调用`resume()`，开始task。   

运行你的app，搜索一些歌曲，你将看到表视图显示了和搜索内容相关的结果。如下所示：     

![](https://cdn2.raywenderlich.com/wp-content/uploads/2015/08/Simulator-Screen-Shot-12-Aug-2015-11.02.34-pm-281x500.png)    

因为添加了一些magic `NSURLSession`代码，Half Tunes现在总算是有点功能了。    

## 下载搜索结果   

可以看到歌曲的搜索结果是很nice，但是如果可以点击下载的话，是不是会更加nice，这是我们接下来该做的事。   

为了轻轻松松的执行多个下载，首先要创建一个类，去控制下载任务的状态。   

创建一个新的文件，命名为`Download.swift`在`Data Objects`文件夹中。   

打开`Download.swift`添加下面的实现：   

```swift
class Download: NSObject {
 
  var url: String
  var isDownloading = false
  var progress: Float = 0.0
 
  var downloadTask: NSURLSessionDownloadTask?
  var resumeData: NSData?
 
  init(url: String) {
    self.url = url
  }
}
```

下面是`Download`这个类的属性的解释：   

* `url`:被下载文件的URL，这也是不同的`Download`的唯一标识符。    
* `isDownloading`:是否正在下载。
* `progress`:下载进度，float类型，0.0 - 1.0。
* `downloadTask`:`NSURLSessionDownloadTask`用来下载任务。
* `resumeData`:在暂停下载的时候，保存已经下载的数据。用来实现断点续传。   

切换到`SearchViewController.swift `，在类的顶部添加如下代码：   

```
var activeDownloads = [String: Download]()
```

这样就可以很简单的通过一个字典来标识`url`对应的下载任务    

## 创建一个下载任务
根据我们的任务清单，现在要做的是实现文件的下载功能，首先，要创建一个专用的session去处理下载任务。    

在`SearchViewController.swift`文件中，`viewDidLoad():`方法之前的合适的位置添加下面的代码：     
```swift
lazy var downloadsSession: NSURLSession = {
  let configuration = NSURLSessionConfiguration.defaultSessionConfiguration()
  let session = NSURLSession(configuration: configuration, delegate: self, delegateQueue: nil)
  return session
}()
```
我们使用默认的`defaultSessionConfiguration`创建了一个单独的session来处理所有的下载任务。同时指定一个`delegate`，用来接收`NSURLSession`的回调，比如：完成回调，下载进度回调等。    

`delegateQueue`设置为`nil`，会为我们创建一个默认的串行队列执行所有的代理回调和完成block的回调。   

注意，懒加载`downloadsSession:`会让我们在需要的时候再创建session，更值得注意的是，我们将传`self`作为代理的参数初始化session，即使`self`没有被初始化。   

在`SearchViewController.swift`文件中，添加` NSURLSessionDownloadDelegate`扩展：   
```swift
extension SearchViewController: NSURLSessionDownloadDelegate {
  func URLSession(session: NSURLSession, downloadTask: NSURLSessionDownloadTask, didFinishDownloadingToURL location: NSURL) {
    print("Finished downloading.")
  }
}
```
`NSURLSessionDownloadDelegate`定义了当你使用`NSURLSessionDownloadTask`的时候要实现的代理方法，唯一的必须实现的方法为：`URLSession(_:downloadTask:didFinishDownloadingToURL:)`，每当下载完成的时候回调用它，每当下载完成的时候，你可以实现这个方法来打印一些简单的信息。   

完成session和代理的配置，接下来我们要准备创建一个下载任务，当用户点击下载的时候。   

在`SearchViewController.swift`文件中,用下面的代码替换`startDownload(_:) `的实现代码：    
```swift
func startDownload(track: Track) {
  if let urlString = track.previewUrl, url =  NSURL(string: urlString) {
    // 1
    let download = Download(url: urlString)
    // 2
    download.downloadTask = downloadsSession.downloadTaskWithURL(url)
    // 3
    download.downloadTask!.resume()
    // 4
    download.isDownloading = true
    // 5
    activeDownloads[download.url] = download
  }
}
```
当用户点击下载按钮的时候，就会调用上面的方法，下载相应的内容，按照代码中的数字标记来看：

&emsp;1.首先使用`track`的`previewURL`实例化一个`Download`对象。    
&emsp;2.然后用刚才懒加载的session创建一个`NSURLSessionDownloadTask`，并将它赋值给`Download`的`downloadTask`属性。     
&emsp;3.调用`resume()`方法开始下载任务。     
&emsp;4.将`isDownloading`设置为`true`标记正在下载。     
&emsp;5.以`url`为键，`download`为值，存入`activeDownloads`字典中。      

运行你的app，输入一个关键字搜索，然后点击`cell`上面的`Download`按钮，过一会就会看到控制台上打印的信息：`Finished downloading.`，代表下载完成了。    
## 保存并且播放
当一个下载任务完成的时候，` URLSession(_:downloadTask:didFinishDownloadingToURL:)`代理方法会提供一个临时的路径保存下载的数据，我们的工作是把它移动带app的沙盒中的不变位置，并且把这个下载任务从`activeDownloads`字典中移除。   

我们需要添加一些帮助方法让事情更轻松一些，在`SearchViewController.swift`类中，添加如下方法：
```swift
func trackIndexForDownloadTask(downloadTask: NSURLSessionDownloadTask) -> Int? {
  if let url = downloadTask.originalRequest?.URL?.absoluteString {
    for (index, track) in searchResults.enumerate() {
      if url == track.previewUrl! {
        return index
      }
    }
  }
  return nil
}
```
每一个`track`都有一个唯一的`url`标识，这个方法仅仅返回了给定的`url`标记的`track`在`searchResults`数组中的`index`。   

然后，用下面的代码代替`URLSession(_:downloadTask:didFinishDownloadingToURL:) `方法：
```swift
func URLSession(session: NSURLSession, downloadTask: NSURLSessionDownloadTask, didFinishDownloadingToURL location: NSURL) {
  // 1
  if let originalURL = downloadTask.originalRequest?.URL?.absoluteString,
    destinationURL = localFilePathForUrl(originalURL) {
 
    print(destinationURL)
 
    // 2
    let fileManager = NSFileManager.defaultManager()
    do {
      try fileManager.removeItemAtURL(destinationURL)
    } catch {
      // Non-fatal: file probably doesn't exist
    }
    do {
      try fileManager.copyItemAtURL(location, toURL: destinationURL)
    } catch let error as NSError {
      print("Could not copy file to disk: \(error.localizedDescription)")
    }
  }
 
  // 3
  if let url = downloadTask.originalRequest?.URL?.absoluteString {
    activeDownloads[url] = nil
    // 4
    if let trackIndex = trackIndexForDownloadTask(downloadTask) {
      dispatch_async(dispatch_get_main_queue(), {
        self.tableView.reloadRowsAtIndexPaths([NSIndexPath(forRow: trackIndex, inSection: 0)], withRowAnimation: .None)
      })
    }
  }
}
```
关键步骤：    

&emsp;1.获取`task`的`url`，并且把它作为`localFilePathForUrl(_:)`的参数。`localFilePathForUrl(_:)`把传入的`url`作为文件名拼接到沙盒的`Documents`路径。    
&emsp;2.在把数据从临时路径拷贝到期望的目标路径之前，使用`NSFileManager`把目标路径中已经存在的缓存删除掉。     
&emsp;3.在完成拷贝之后，找到字典中相应的task，把它设置为`nil`。    
&emsp;4.找到表视图中的显示这首歌曲的`cell`，更新它。   

运行你的app，搜索一些内容并且下载它，你将会在控制台看到打印的路径信息。  

`Download`按钮也会隐藏，这首歌曲现在在你的设备中，点击`cell`，就可以跳转到`MPMoviePlayerViewController`播放，如下所示：   

![](https://cdn2.raywenderlich.com/wp-content/uploads/2015/08/Simulator-Screen-Shot-17-Aug-2015-1.45.28-am-281x500.png)         
##监听下载进度
到目前为止，我们没有做任何事监听下载进度，为了提高用户体验，我们将做一些事监听下载进度，并且显示到`cell`上。  
打开`SearchViewController.swift`找到`NSURLSessionDownloadDelegate`的扩展，实现下面的代理方法：   
```swift
func URLSession(session: NSURLSession, downloadTask: NSURLSessionDownloadTask, didWriteData bytesWritten: Int64, totalBytesWritten: Int64, totalBytesExpectedToWrite: Int64) {
 
    // 1
    if let downloadUrl = downloadTask.originalRequest?.URL?.absoluteString,
      download = activeDownloads[downloadUrl] {
      // 2
      download.progress = Float(totalBytesWritten)/Float(totalBytesExpectedToWrite)
      // 3
      let totalSize = NSByteCountFormatter.stringFromByteCount(totalBytesExpectedToWrite, countStyle: NSByteCountFormatterCountStyle.Binary)
      // 4
      if let trackIndex = trackIndexForDownloadTask(downloadTask), let trackCell = tableView.cellForRowAtIndexPath(NSIndexPath(forRow: trackIndex, inSection: 0)) as? TrackCell {
        dispatch_async(dispatch_get_main_queue(), {
          trackCell.progressView.progress = download.progress
          trackCell.progressLabel.text =  String(format: "%.1f%% of %@",  download.progress * 100, totalSize)
        })
    }
  }
}
```
一步一步的来看这个代理方法：   

&emsp;1.用方法提供的`downloadTask`参数，获取它的`url`，利用`url`在`activeDownloads`字典中取出相应的`Download`。     
&emsp;2.这个方法也会返回已经下载的字节数和文件的总字节数，我们将计算这两个数的比值，赋值给`Download`的`progress`属性，我们将用这个值来更新`cell`上的`progress view`。     
&emsp;3.调用了`NSByteCountFormatter`的类方法将文件的内存大小从字节数转化为人类可读的字符串，在`progress label`上显示。   
&emsp;4.最后，找到和这首歌曲对应的`cell`，回到主线程更新他的`progress view`和`progress label`。     

然后，当下载进行中的时候，配置`cell`，显示相应的下载进度和状态。    

在`tableView(_:cellForRowAtIndexPath:):`方法中，找到：
```swift
let downloaded = localFileExistsForTrack(track)
```
在这行代码的前面添加：
```swift
var showDownloadControls = false
if let download = activeDownloads[track.previewUrl!] {
  showDownloadControls = true
 
  cell.progressView.progress = download.progress
  cell.progressLabel.text = (download.isDownloading) ? "Downloading..." : "Paused"
}
cell.progressView.hidden = !showDownloadControls
cell.progressLabel.hidden = !showDownloadControls
```
添加了一个布尔类型的变量`showDownloadControls`，作为标记来更新`progress view`和`progress label`的显示内容和是否隐藏。   

当暂停下载的时候显示`Paused`，正在下载的时候显示`Downloading...`。    

然后，把下面一行代码：    
```swift
cell.downloadButton.hidden = downloaded
```
替换为：     
```swift
cell.downloadButton.hidden = downloaded || showDownloadControls
```
当我们正在下载的时候，把`Download`按钮隐藏。    

运行你的app，随便下载一首歌曲，就会看到我们已经完成了对下载进度的监听以及相关状态的改变。    

## 暂停、继续、取消下载
假如用户需要暂停或者取消下载会怎么样？在这一部分，我们将实现下载的暂停、继续和取消功能。    

我们从允许用户取消正在下载的任务开始。    

替换`cancelDownload(_:)`方法为：
```swift
func cancelDownload(track: Track) {
  if let urlString = track.previewUrl,
    download = activeDownloads[urlString] {
      download.downloadTask?.cancel()
      activeDownloads[urlString] = nil
  }
}
```
我们从`activeDownloads`字典中取出相应的`download`，然后调用`cancel()`方法，取消这个任务。然后把这个`download`从字典中移除。    

暂停下载和取消下载在概念上是非常相似的，不同的地方是，暂停下载的会生成`resume data`，`resume data`包含接下来继续任务时的信息，需要服务端提供此功能。    

用下面的代码替换`pauseDownload(_:) `方法：
```swift
func pauseDownload(track: Track) {
  if let urlString = track.previewUrl,
    download = activeDownloads[urlString] {
      if(download.isDownloading) {
        download.downloadTask?.cancelByProducingResumeData { data in
          if data != nil {
            download.resumeData = data
          }
        }
        download.isDownloading = false
      }
  }
}
```
和取消方法的不同点是，这里调用了`cancelByProducingResumeData(_:) `方法，而不是`cancel()`。我们在暂停下载的时候，生成了`resume data`,并且取出赋值给`download`的`resumeData`属性。    

同时设置`isDownloading`为`false`标记下载暂停。    

接下来要做的是从暂停的地方继续下载任务。     

用下面的代码代替`resumeDownload(_:)`方法：
```swift
func resumeDownload(track: Track) {
  if let urlString = track.previewUrl,
    download = activeDownloads[urlString] {
      if let resumeData = download.resumeData {
        download.downloadTask = downloadsSession.downloadTaskWithResumeData(resumeData)
        download.downloadTask!.resume()
        download.isDownloading = true
      } else if let url = NSURL(string: download.url) {
        download.downloadTask = downloadsSession.downloadTaskWithURL(url)
        download.downloadTask!.resume()
        download.isDownloading = true
      }
  }
}
```
我们首先判断，要继续的任务的`resumeData`是否为`nil`，以便于决定是从断点处继续下载，还是从头开始下载。如果不为空，则调用`downloadTaskWithResumeData(_:)`方法、使用`resume data`创建一个新的任务，并且调用`resume()`执行任务。如果`resume data`为空，则创建一个新的任务，传入`url`，重新开始下载。    

在这两种情况下，都要把`isDownloading`设置为`true`，标记为正在下载。    

接下来还有一件事要去做，就是在合适的时间显示和隐藏`Pause`、`Cancel`和`Resume`按钮。    

找到`tableView(_:cellForRowAtIndexPath:)`方法，找到下面的代码：
```swift
if let download = activeDownloads[track.previewUrl!] {
```
在条件语句的最后加入下面两行代码：
```swift
let title = (download.isDownloading) ? "Pause" : "Resume"
cell.pauseButton.setTitle(title, forState: UIControlState.Normal)
```
这两行代码的作用就是让按钮的`titleLabel`在正确的状态显示正确的标识(`Pause`or`Resume`)。    

接下来，在`reture`之前加入下面的代码：   
```swift
cell.pauseButton.hidden = !showDownloadControls
cell.cancelButton.hidden = !showDownloadControls
```
这两句就是实现了是否显示这两个按钮。   

运行你的app，同时下载几首歌曲，你可以暂停、继续和取消他们，like this：    
![](https://cdn1.raywenderlich.com/wp-content/uploads/2015/08/Simulator-Screen-Shot-18-Aug-2015-10.14.38-pm-281x500.png)  

## 实现后台下载
现在，我们的app已经初具雏形了，但是，我们还需要添加一些更优雅的东西使用户体验更好。如果因为一些原因app进入后台或者崩溃的时候，让正在下载的任务继续下载。   

如果我们的app没有运行，怎么能继续下载工作呢？有一个单独的后台进程在app的外部运行，并且管理后台的传输任务。在app运行的时候，它会向app发送合适的代理信息。当app在运行的时候突然中止，任务会在后台继续进行。   

当任务完成的时候，后台进程会在后台重新打开app。重新打开的app会重新链接和之前相同的会话、接收相关的完成代理信息和执行一些动作，比如：存储文件到本地磁盘。   

仍然打开` SearchViewController.swift`，在`downloadsSession`的初始化中，找到下面的代码：
```swift
let configuration = NSURLSessionConfiguration.defaultSessionConfiguration()
```
用下面的代码替换它：
```swift
let configuration = NSURLSessionConfiguration.backgroundSessionConfigurationWithIdentifier("bgSessionConfiguration")
```
将默认的session配置用特殊的`backgroundSessionConfiguration`替换，并且为它设置一个identifier，目的是为了实现上文中提到的重新连接会话。   

然后在`viewDidLoad()`，添加下面代码：
```swift
_ = self.downloadsSession
```
这行代码的作用是保证`SearchViewController`初始化的同时懒加载一个`downloadsSession`对象。   

当后台的任务完成的时候，而app没有在运行，app将在后台重新启动，我们将在app delegate里面完成这个操作。   

切换到`AppDelegate.swift`，在这个类的顶部添加下面代码：
```swift
var backgroundSessionCompletionHandler: (() -> Void)?
```
然后添加下面的方法：
```swift
func application(application: UIApplication, handleEventsForBackgroundURLSession identifier: String, completionHandler: () -> Void) {
  backgroundSessionCompletionHandler = completionHandler
}
```
现在，我们在app delegate中保存了一个`completionHandler`block变量，以便后面使用。    

`application(_:handleEventsForBackgroundURLSession:) `方法会唤醒app去处理完成的后台任务，我们需要在这个事件处理两个东西：  
&emsp;1.首先，app需要通过代理方法提供的标识符(identifier)重新链接相关的后台会话，但是因为每次创建和使用后台会话的时候都要实例化`SearchViewController`，此时就已经重新链接了。     
&emsp;2.我们需要捕获代理方法提供的完成回调block，调用完成回调block使系统把更新的UI快照显示在app切换器上，同时，告诉系统关于当前会话的所有后台活动都已经完成。   

但是，我们什么时候调用这个完成处理block呢？   

`URLSessionDidFinishEventsForBackgroundURLSession(_:)`方法会是一个好的选择，它是`NSURLSessionDelegate`代理方法，当所有的后台会话完成的时候会被调用。    

在`SearchViewController.swift`中实现这个方法：
```swift
extension SearchViewController: NSURLSessionDelegate {
 
  func URLSessionDidFinishEventsForBackgroundURLSession(session: NSURLSession) {
    if let appDelegate = UIApplication.sharedApplication().delegate as? AppDelegate {
      if let completionHandler = appDelegate.backgroundSessionCompletionHandler {
        appDelegate.backgroundSessionCompletionHandler = nil
        dispatch_async(dispatch_get_main_queue(), {
          completionHandler()
        })
      }
    }
  }
}
```
上述代码抓取了存在app delegate中的完成回调block，并且在主线程调用它。    

运行你的app。同时开始几个下载任务，然后按下`Home`键，使app在后台运行，等到你认为下载任务完成的时候，双击`Home`键，显示app切换器。   

下载任务应该已经完成了，并且可以在屏幕上看到新的关于下载完成的状态：    
![](https://cdn5.raywenderlich.com/wp-content/uploads/2015/08/Simulator-Screen-Shot-19-Aug-2015-1.06.24-am-281x500.png)    
现在，你拥有了一个完整功能的音乐流媒体app，移动的` Apple Music`！:]   

## 接下来该做什么
你可以在[这里](http://www.raywenderlich.com/wp-content/uploads/2016/01/HalfTunes-Final.zip)下载本教程完整的项目。   

恭喜，你现在可以很好的处理你的app中一般网络请求，当然还有很多比本教程更详细的`NSURLSession`应用，比如，上传任务，设置会话的配置（超时，缓存策略等）。   

通过如下资源学习更多的特性吧：   
*Apple的[官方文档](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/)包含了所有API提供的方法的详细信息。     
*我们自己的书：[iOS7教程](http://www.raywenderlich.com/store/ios-7-by-tutorials)，其中包含了两个章节专门讲解`NSURLSession`，你也可以阅读我们的[NSURLSession教程](http://www.raywenderlich.com/51127/nsurlsession-tutorial)。
*[AlamoFire](https://github.com/Alamofire/Alamofire)也是非常流行的第三方网络库，我们在[Begining AlamoFire](http://www.raywenderlich.com/85080/beginning-alamofire-tutorial)教程中讲解了它的基础内容。   

希望这篇教程能对你有用，参与下方讨论吧！    

-----------
译者注：欢迎转载，但请一定要注明出处！谢谢！
