So you've been developing [Ionic Framework](http://www.ionicframework.com) hybrid Android and iOS applications for a while now and even started saving media to the file system such as images or videos.  However, now that you've saved this media, what do you do about finding it?

From popular request on my [Twitter channel](https://www.twitter.com/nraboy), we're going to see how to create a file browser in Ionic Framework that will display all files and directories and let you traverse deeper into the device to find what you're looking for.  This is useful for determining file paths in case you need to load or manage them.

## Creating a New Ionic Framework Project

### The Requirements

* NPM, Apache Cordova, Ionic, and Android installed and configured
* The [Apache Cordova File](https://cordova.apache.org/docs/en/3.0.0/cordova_file_file.md.html) plugin

### Creating Our Project

To start things off, we're going to create a fresh Ionic project using our Terminal (Mac and Linux) or Command Prompt (Windows):

```
ionic start FileBrowserApp blank
cd FileBrowserApp
ionic platform add android
ionic platform add ios
```

Remember, if you're not using a Mac with Xcode installed, you cannot add and build for the iOS platform.

At this point we're left with a new Android and iOS project using the Ionic Framework blank template.

### Installing The Dependencies

This application requires access to the filesystem which can only be done through a plugin.  To be specific, the [Apache Cordova File](https://cordova.apache.org/docs/en/3.0.0/cordova_file_file.md.html) plugin.  This plugin can be installed from the Terminal or Command Prompt like the following:

```
cordova plugin add https://git-wip-us.apache.org/repos/asf/cordova-plugin-file.git
```

I've written a few [tutorials that make use of the File plugin](https://blog.nraboy.com/2014/09/manage-files-in-android-and-ios-using-ionicframework/), but what we'll see here is a bit different.  Regardless, it is recommended that you read the official documentation.

## Adding The Code That Matters

### Creating a File Factory

To manage file operations we're going to create an [AngularJS](https://angularjs.org/) factory.  In this factory, we're going to have three functions:

* A `getParentDirectory(path)` function for determining the parent directory of the current path
* A `getEntriesAtRoot()` function for getting all files and directories at the root of the device
* A `getEntries(path)` method that will get all files and directories at a given path

Open your project's **www/js/app.js** file and add the following factory:

```javascript
.factory("$fileFactory", function($q) {

    var File = function() { };

    File.prototype = {

        getParentDirectory: function(path) {
            var deferred = $q.defer();
            window.resolveLocalFileSystemURI(path, function(fileSystem) {
                fileSystem.getParent(function(result) {
                    deferred.resolve(result);
                }, function(error) {
                    deferred.reject(error);
                });
            }, function(error) {
                deferred.reject(error);
            });
            return deferred.promise;
        },

        getEntriesAtRoot: function() {
            var deferred = $q.defer();
            window.requestFileSystem(LocalFileSystem.PERSISTENT, 0, function(fileSystem) {
                var directoryReader = fileSystem.root.createReader();
                directoryReader.readEntries(function(entries) {
                    deferred.resolve(entries);
                }, function(error) {
                    deferred.reject(error);
                });
            }, function(error) {
                deferred.reject(error);
            });
            return deferred.promise;
        },

        getEntries: function(path) {
            var deferred = $q.defer();
            window.resolveLocalFileSystemURI(path, function(fileSystem) {
                var directoryReader = fileSystem.createReader();
                directoryReader.readEntries(function(entries) {
                    deferred.resolve(entries);
                }, function(error) {
                    deferred.reject(error);
                });
            }, function(error) {
                deferred.reject(error);
            });
            return deferred.promise;
        }

    };

    return File;

});
```

Is this the best way to get the job done?  Probably not, but it should give you a good idea to take it to the next level.  We make use of AngularJS promises because all file events are asynchronous so we have to plan appropriately.  Beyond that, the functions don't stray far from the official Apache Cordova documentation.

### Creating Our View

What we see on the screen is going to be very simple.  We will add all files and directories to a list view and show a specific icon depending on if the list item is a file or a directory.  If the item is a file we will show a document icon, otherwise we will show a folder icon.

Open your project's **www/index.html** file and replace the already existing `<ion-content>` tags with the following:

```markup
<ion-content ng-controller="ExampleController">
    <div class="list">
        <div ng-repeat="file in files">
            <a class="item item-icon-left" href="#" ng-click="getContents(file.nativeURL)">
                <i ng-show="file.isDirectory" class="icon ion-folder"></i>
                <i ng-show="file.isFile" class="icon ion-document"></i>
                {{file.name}}
            </a>
        </div>
    </div>
</ion-content>
```

Hold on a minute.  What is this `getContents` method, or `ExampleController` AngularJS controller?  We'll get there in a minute.

### The Logic To Power Our View

The plan here is to have a single view application.  With that said, when the application loads it will get all files and directories at the root of the device.  From within the view, if any directory within the list is clicked, the list will be replaced with all entries at the desired path.  This is accomplished through the `getContents(path)` method.

Open your project's **www/js/app.js** file and add the following controller:

```javascript
.controller("ExampleController", function($scope, $ionicPlatform, $fileFactory) {

    var fs = new $fileFactory();

    $ionicPlatform.ready(function() {
        fs.getEntriesAtRoot().then(function(result) {
            $scope.files = result;
        }, function(error) {
            console.error(error);
        });

        $scope.getContents = function(path) {
            fs.getEntries(path).then(function(result) {
                $scope.files = result;
                $scope.files.unshift({name: "[parent]"});
                fs.getParentDirectory(path).then(function(result) {
                    result.name = "[parent]";
                    $scope.files[0] = result;
                });
            });
        }
    });

})
```

You'll notice in the `getContents(path)` method that we create a dummy **[parent]** element at the top of the list.  This will later be replaced with information about the parent directory which will allow us to traverse backwards instead of only forward.

## Testing Our New Application

### Using a Web Browser

Don't do it.  If you even attempt to open this project in a web browser you're setting yourself up for failure.  This application uses native device plugins that the web browser is unfamiliar with.  In turn this will give strangeness and errors.

### Using a Device or Simulator

To test this application on your device or simulator, run the following in your Command Prompt or Terminal:

```
ionic build android
adb install -r platforms/android/ant-build/MainActivity-debug.apk
```

The above will get you going on Android. If you wish to track down errors and view the logs, you can use the Android Debug Bridge (ADB) like so:

```
adb logcat
```

More information on this can be seen in one of my [other tutorials](https://blog.nraboy.com/2014/12/debugging-android-source-code-adb/) on the topic.

## Conclusion

You've just made a very simple file browser using Apache Cordova and Ionic Framework.  It can access the device file system and display as well as traverse any files and folders.

What did I leave to the imagination?  It is up to you what you'd like to do when clicking a file versus a directory.
