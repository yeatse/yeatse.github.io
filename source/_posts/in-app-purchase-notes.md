---
title: IAP文档笔记
tags:
  - in-app purchase
  - iOS
id: 224
categories:
  - 笔记
date: 2016-02-17 19:55:03
---

# IAP(In-App Purchase) Programming Notes
[iOS Dev Lib Reference](
https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Introduction.html)

[TOC]

## At a Glance
![States of the purchase process](/images/2016/intro_2x.png)

<!-- more -->

## iTunes Connect Configurations
* Configure IAP products.
* Test IAP products.
* Submit IAP products for review.
* Manage the IAP products available in the app.
* [iOS dev lib reference](https://developer.apple.com/library/ios/documentation/LanguagesUtilities/Conceptual/iTunesConnectInAppPurchase_Guide/Chapters/Introduction.html)

## Retrieving Product Information
![displaying store UI](/images/2016/retrieving_product_info_2x.png)

* Get a list of product identifiers.
    * Product ids can be retrieved through network or app bundle.
    * Product ids are configured in iTunes Connect, and formed like: "com.example.level1", "com.example.rocket_car".

* Retrieving product information.
    * Create a instance of [SKProductsRequest](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKProductsRequest/index.html) and initialize it with a list of product ids.
    * The `SKProductsRequest` should be kept with a strong reference in case of instant deallocation.
    * Results -- both valid and invalid ones -- are called back to the delegate.
    * Valid products are encapsulated as a list of `SKProduct` objects, from which we can get the titles/descriptions/prices.

* Presenting the app's store ui.
    * Users can be restricted to access the store payment. Call [canMakePayments](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentQueue_Class/index.html#//apple_ref/occ/clm/SKPaymentQueue/canMakePayments) class method (`SKPaymentQueue`) before displaying the store ui or an error message.
    * Users should be informed exactly what they are going to buy.
    * Display prices clearly, using the locale and currency returned by the App Store.
    * A product's price formatting example:
    
    ```objectivec
    NSNumberFormatter *numberFormatter = [[NSNumberFormatter alloc] init];
    [numberFormatter setFormatterBehavior:NSNumberFormatterBehavior10_4];
    [numberFormatter setNumberStyle:NSNumberFormatterCurrencyStyle];
    [numberFormatter setLocale:product.priceLocale];
    NSString *formattedPrice = [numberFormatter stringFromNumber:product.price];
    ```
* Suggested testing steps:
    [iOS Dev Lib Ref](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/ShowUI.html#//apple_ref/doc/uid/TP40008267-CH3-SW11)

## Requesting Payment
![requesting_payment_2x](/images/2016/requesting_payment_2x.png)

* Creating a payment request.
    
    ```objectivec
    SKProduct *product = <# Product returned by a products request #>;
    SKMutablePayment *payment = [SKMutablePayment paymentWithProduct:product];
    payment.quantity = 2;
    ```

* [applicationUsername](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentRequest_Class/index.html#//apple_ref/occ/instp/SKPayment/applicationUsername) of the `SKMutablePayment` object can be populated with a one-way hash of the user's app account name for further authorization.

* Submitting a payment request.
    
    ```objectivec
    [[SKpaymentQueue defaultQueue] addPayment:payment];
    ```

## Delivering Products
![delivering products](/images/2016/delivering_products_2x.png)

* Waiting for the App Store to process transations.
    * Central role: *transaction queue*
    * Payment requests are added to the transaction queue.
    * When the transation's state changes, Store Kit calls the app's *transaction queue observer*.
    * The observer must conform to the [SKPaymentTransactionObserver](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransactionObserver_Protocol/index.html) protocol.
    * Register the observer at launch time, and make sure that the observer is ready to handle a transaction at any time.
        
        ```objectivec
        - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
        {
            /* ... */
            
            [[SKPaymentQueue defaultQueue] addTransactionObserver:observer];
        }
        ```
    * If your app fails to mark a transaction as finished, Store Kit calls the observer every time your app is launched until the transaction is properly finished.
    * Store Kit call the [paymentQueue:updatedTransactions:](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransactionObserver_Protocol/index.html#//apple_ref/occ/intfm/SKPaymentTransactionObserver/paymentQueue:updatedTransactions:) method when the status of a transaction changes.
    * The transaction status tells you what action your app needs to perform:
        
        Status | Action to take in your app
        ------ | ------
        [SKPaymentTransactionStatePurchasing](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/index.html#//apple_ref/c/econst/SKPaymentTransactionStatePurchasing) | Update your UI to reflect the in-progress status, and wait to be called again.
        [SKPaymentTransactionStateDeferred](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/index.html#//apple_ref/c/econst/SKPaymentTransactionStateDeferred) | Update your UI to reflect the deferred status, and wait to be called again.
        [SKPaymentTransactionStateFailed](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/index.html#//apple_ref/c/econst/SKPaymentTransactionStateFailed) | Use the value of the error property to present a message to the user. For a list of error constants, see [SKErrorDomain](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/StoreKitTypes/index.html#//apple_ref/doc/constant_group/SKErrorDomain) in [StoreKit Constants Reference](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/StoreKitTypes/index.html#//apple_ref/doc/uid/TP40008588).
        [SKPaymentTransactionStatePurchased](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/index.html#//apple_ref/c/econst/SKPaymentTransactionStatePurchased) | Provide the purchased functionality.
        [SKPaymentTransactionStateRestored](https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/index.html#//apple_ref/c/econst/SKPaymentTransactionStateRestored) | Restore the previously purchased functionality.
    
    * Responding to transaction statuses:
        
        ```objectivec
        - (void)paymentQueue:(SKPaymentQueue *)queue
         updatedTransactions:(NSArray *)transactions
        {
            for (SKPaymentTransaction *transaction in transactions) {
                switch (transaction.transactionState) {
                    // Call the appropriate custom method for the transaction state.
                    case SKPaymentTransactionStatePurchasing:
                        [self showTransactionAsInProgress:transaction deferred:NO];
                        break;
                    case SKPaymentTransactionStateDeferred:
                        [self showTransactionAsInProgress:transaction deferred:YES];
                        break;
                    case SKPaymentTransactionStateFailed:
                        [self failedTransaction:transaction];
                        break;
                    case SKPaymentTransactionStatePurchased:
                        [self completeTransaction:transaction];
                        break;
                    case SKPaymentTransactionStateRestored:
                        [self restoreTransaction:transaction];
                        break;
                    default:
                        // For debugging
                        NSLog(@"Unexpected transaction state %@", @(transaction.transactionState));
                        break;
                }
            }
        }
        ```
    * Implement `paymentQueue:removedTransactions:` method to remove the corresponding items from app's UI.
    * Implement `paymentQueueRestoreCompletedTransactionsFinished:` or `paymentQueue:restoreCompletedTransactionsFailedWithError:` method to reflect the restoring result.

* Finishing the transaction
    * Your app needs to finish every transaction, regardless of whether the transaction succeeded or failed.
    * Complete all of the following actions before finishing the transaction:
        1. [Persisting the purchase](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/DeliverProduct.html#//apple_ref/doc/uid/TP40008267-CH5-SW5)
        2. [Download associated content](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/DeliverProduct.html#//apple_ref/doc/uid/TP40008267-CH5-SW9)
        3. [Update UI to unlock the app functionality](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/DeliverProduct.html#//apple_ref/doc/uid/TP40008267-CH5-SW20)
    * To finish a transaction, call the `finishTransaction:` method:
        
        ```objectivec
        SKPaymentTransaction *transaction = <# The current payment #>;
        [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
        ```
    * Suggested testing steps:
        [iOS Dev Lib Ref](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/DeliverProduct.html#//apple_ref/doc/uid/TP40008267-CH5-SW12)
                
## Preparing for App Review
[iOS Dev Lib Ref](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/AppReview.html#//apple_ref/doc/uid/TP40008267-CH10-SW1)

