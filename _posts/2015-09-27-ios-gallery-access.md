---
layout: post
title: 在iOS上选取图库图片并保存图片到图库
tags:
  - iOS
id: 174
categories:
  - 笔记
date: 2015-09-27 00:43:45 +0800
---

好久没发东西了，人太懒。。。

标题是公司项目里要实现的功能，其实没啥难度。但是从iOS9.0开始`ALAssetsLibrary`被废弃了，由`PHPhotoLibrary`替代了，所以网上的方法不能用了。自己查文档查stackoverflow，重新实现了功能，然后简单封装了一下，留个记录。iOS7以上系统可用。

<!-- more -->

GalleryAccess.h：

```objectivec
//
//  GalleryAccess.h
//  bukaios
//
//  Created by yeatse on 15/9/23.
//  Copyright © 2015年 bukaios. All rights reserved.
//

#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

@interface GalleryAccess : NSObject

+ (void)pickImageFromController:(UIViewController*)controller
                        handler:(void(^)(UIImage* result))handler;

+ (void)saveImage:(UIImage*)image
          toAlbum:(NSString*)albumName
          handler:(void(^)(NSError* error))handler;

@end
```

GalleryAccess.m

```objectivec
//
//  GalleryAccess.m
//  bukaios
//
//  Created by yeatse on 15/9/23.
//  Copyright © 2015年 bukaios. All rights reserved.
//

#import "GalleryAccess.h"
#import <MobileCoreServices/MobileCoreServices.h>
#import <AssetsLibrary/AssetsLibrary.h>
#import <Photos/Photos.h>

#define IMPLEMENT_SINGLETON(funcname, clsname) \
+ (id)funcname \
{ \
static id i = nil; \
static dispatch_once_t once; \
dispatch_once(&once, ^{ \
i = [[self alloc] init]; \
}); \
return i; \
}

@interface GalleryAccess()<UINavigationControllerDelegate, UIImagePickerControllerDelegate>

@property (strong) void(^pickImageHandler)(UIImage* result);

@end

@implementation GalleryAccess

IMPLEMENT_SINGLETON(instance, GalleryAccess);

+ (void)pickImageFromController:(UIViewController *)controller handler:(void (^)(UIImage *))handler
{
    GalleryAccess* mgr = [GalleryAccess instance];
    mgr.pickImageHandler = handler;

    UIImagePickerController* picker = [[UIImagePickerController alloc] init];
    picker.delegate = mgr;
    picker.allowsEditing = YES;
    picker.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;

    [controller presentViewController:picker animated:YES completion:NULL];
}

+ (void)saveImage:(UIImage *)image toAlbum:(NSString *)albumName handler:(void (^)(NSError *))handler
{
    GalleryAccess* mgr = [GalleryAccess instance];

    if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 8)
    {
        __block PHAssetCollection* collection;
        __block PHObjectPlaceholder* placeholder;

        PHFetchOptions* fetchOptions = [[PHFetchOptions alloc] init];
        fetchOptions.predicate = [NSPredicate predicateWithFormat:@"localizedTitle = %@", albumName];

        PHFetchResult* fetchResult = [PHAssetCollection fetchAssetCollectionsWithType:PHAssetCollectionTypeAlbum subtype:PHAssetCollectionSubtypeAny options:fetchOptions];
        collection = fetchResult.firstObject;

        if (!collection)
        {
            [[PHPhotoLibrary sharedPhotoLibrary] performChanges:^{
                PHAssetCollectionChangeRequest* createAlbum = [PHAssetCollectionChangeRequest creationRequestForAssetCollectionWithTitle:albumName];
                placeholder = [createAlbum placeholderForCreatedAssetCollection];
            } completionHandler:^(BOOL success, NSError * _Nullable error) {
                if (success)
                {
                    PHFetchResult* collectionFetchResult = [PHAssetCollection fetchAssetCollectionsWithLocalIdentifiers:@[placeholder.localIdentifier] options:nil];
                    collection = collectionFetchResult.firstObject;
                    [mgr addNewAssetWithImage:image toAlbum:collection handler:handler];
                }
                else
                {
                    handler(error);
                }
            }];
        }
        else
        {
            [mgr addNewAssetWithImage:image toAlbum:collection handler:handler];
        }
    }
    else
    {
        ALAssetsLibrary* library = [[ALAssetsLibrary alloc] init];
        [library writeImageToSavedPhotosAlbum:image.CGImage
                                  orientation:(ALAssetOrientation)image.imageOrientation
                              completionBlock:^(NSURL *assetURL, NSError *error) {
                                  if (error)
                                  {
                                      handler(error);
                                  }
                                  else
                                  {
                                      [mgr addAssetURL:assetURL toAlbum:albumName
                                               library:library handler:handler];
                                  }
                              }];
    }
}

- (void)addNewAssetWithImage:(UIImage*)image toAlbum:(PHAssetCollection*)album handler:(void(^)(NSError*))handler
{
    [[PHPhotoLibrary sharedPhotoLibrary] performChanges:^{
        PHAssetChangeRequest* createAssetRequest = [PHAssetChangeRequest creationRequestForAssetFromImage:image];

        PHAssetCollectionChangeRequest* albumChangeRequest = [PHAssetCollectionChangeRequest changeRequestForAssetCollection:album];

        PHObjectPlaceholder* assetPlaceholder = [createAssetRequest placeholderForCreatedAsset];
        [albumChangeRequest addAssets:@[assetPlaceholder]];
    } completionHandler:^(BOOL success, NSError * _Nullable error) {
        handler(error);
    }];
}

- (void)addAssetURL:(NSURL*)assetURL toAlbum:(NSString*)albumName library:(ALAssetsLibrary*)library handler:(void (^)(NSError *))handler
{
    __block BOOL albumFound = NO;
    [library enumerateGroupsWithTypes:ALAssetsGroupAlbum
                           usingBlock:^(ALAssetsGroup *group, BOOL *stop) {
                               if ([albumName compare:[group valueForProperty:ALAssetsGroupPropertyName]
                                              options:NSCaseInsensitiveSearch] == NSOrderedSame)
                               {
                                   albumFound = YES;

                                   [library assetForURL:assetURL
                                            resultBlock:^(ALAsset *asset) {
                                                [group addAsset:asset];
                                                handler(nil);
                                            } failureBlock:handler];

                                   *stop = YES;
                               }

                               if (!group && !albumFound)
                               {
                                   __weak ALAssetsLibrary* weakLib = library;

                                   [library addAssetsGroupAlbumWithName:albumName
                                                            resultBlock:^(ALAssetsGroup *group) {
                                                                [weakLib assetForURL:assetURL
                                                                         resultBlock:^(ALAsset *asset) {
                                                                             [group addAsset:asset];
                                                                             handler(nil);
                                                                         }
                                                                        failureBlock:handler];
                                                            }
                                                           failureBlock:handler];

                                   *stop = YES;
                               }
                           }
                         failureBlock:handler];
}

#pragma mark UIImagePickerControllerDelegate

- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary<NSString *,id> *)info
{
    UIImage* result = nil;
    NSString* type = [info objectForKey:UIImagePickerControllerMediaType];
    if (CFStringCompare((__bridge_retained CFStringRef) type, kUTTypeImage, 0) == kCFCompareEqualTo)
    {
        result = (UIImage*)[info objectForKey:UIImagePickerControllerEditedImage];
        if (!result)
            result = (UIImage*)[info objectForKey:UIImagePickerControllerOriginalImage];
    }
    [picker dismissViewControllerAnimated:YES completion:NULL];

    if (self.pickImageHandler)
        self.pickImageHandler(result);

    self.pickImageHandler = NULL;
}

- (void)imagePickerControllerDidCancel:(UIImagePickerController *)picker
{
    [picker dismissViewControllerAnimated:YES completion:NULL];

    if (self.pickImageHandler)
        self.pickImageHandler(nil);

    self.pickImageHandler = NULL;
}

@end
```

用法很简单，头文件的声明很清楚了~懒得写注释，反正自己能看懂。。。

