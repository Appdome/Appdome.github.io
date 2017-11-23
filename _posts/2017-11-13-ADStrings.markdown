---
layout: post
title:  "Encrypted User Strings"
subtitle: "How to hint appdome as to which strings should be encrypted"
date:   2017-10-31
author: Dany Zatuchna
---

# Usage

Whenever there's a string in the user's code that he wishes to encrypt, for example:

```swift
var billingAddress = Address(
        street1: "1 Infinite Loop",
        street2: "",
           city: "Cupertino",
          state: "CA",
            zip: "95014")
```

He just need to replace each __secure__ string with `SecString("SecString:...")`:

```swift
var billingAddress = Address(
        street1: SecString("SecString:1 Infinite Loop"),
        street2: "",
           city: SecString("SecString:Cupertino"),
          state: SecString("SecString:CA"),
            zip: SecString("SecString:95014"))
```

The purpose of the `"SecString:"` prefix is to allow the appdome fuser to locate the strings and encrypt them, while the `SecString()` function envelope makes sure that the string's contents are available at runtime to the application.

# Encryption

Following fusion, the strings' contents will be completely encrypted in the binary with no way to decipher them.

For example:
```swift
var paymentMethod = PaymentMethod(
        creditCardNumber: SecString("SecString:1234-123456-1234"),
          expirationDate: Date(),
                     cvv: SecString("SecString:999"))
```

In the above example we have a credit-card number and CVV that we want to encrypt.

Before fusion the strings will be clear:
```
000000010003AC50 aSecstring12341 DCB "SecString:1234-123456-1234",0
000000010003AC6B aSecstring999   DCB "SecString:999",0
```

After fusion, an attacker won't be able to even recognize this as a string:
```
000000010003AC50 DCB "f",0xB0,0xC7,0x3D,0xF4,0x5E,0x56,0x2C
000000010003AC50 DCB "E",0xA3," ",0x1E,"l0",5,"|"
000000010003AC50 DCB 0xC4,0x41,0xDD,0xEB,0xB,"?",0xE7,0x24,0xE0
000000010003AC50 DCB 0xCD,0x8C,3,0x83,0xB3,"SZ7O"
000000010003AC50 DCB 0xA7,6,0xC8,0xD0,0
```

# What needs to be done

It goes without saying that we want the written program to function correctly without it being fused (for testing etc...), therefore we need to add some boilerplate code to the Xcode project so all the weird `SecString("SecString:...")` syntax remain inert.

The following code needs to be added to the Xcode project (preferably in its root folder):

1. `OCSecString.m`:

   ```objc
   #import <Foundation/Foundation.h>
   @interface OCSecString : NSString
   @property (nonatomic, strong) NSString *stringHolder;
   @end

   @implementation OCSecString
   - (instancetype)initWithCharactersNoCopy:(unichar *)characters
                                     length:(NSUInteger)length
                               freeWhenDone:(BOOL)freeBuffer
   {
       self = [super init];
       if (self) {
           if (characters[0] == 'S' &&
               characters[1] == 'e' &&
               characters[2] == 'c' &&
               characters[3] == 'S' &&
               characters[4] == 't' &&
               characters[5] == 'r' &&
               characters[6] == 'i' &&
               characters[7] == 'n' &&
               characters[8] == 'g' &&
               characters[9] == ':') {
               self.stringHolder = [[NSString alloc]
                   initWithCharactersNoCopy:characters
                                     length:length
                               freeWhenDone:freeBuffer];
           }
           else
           {
               NSException *ex = [NSException
                   exceptionWithName:@"SecString format error"
                              reason:@"No \"SecString:\" prefix found"
                            userInfo:nil];
               @throw ex;
           }
       }
       return self;
   }

   - (NSUInteger)length
   {
       return self.stringHolder.length - 10;
   }

   - (unichar)characterAtIndex:(NSUInteger)index
   {
       return [self.stringHolder characterAtIndex:index + 10];
   }
   @end
   ```
2. `OSecString.h`:
   ```objc
   #ifndef SecString_h
   #define SecString_h
   #import <Foundation/Foundation.h>
   @interface OCSecString : NSString
   @end
   #endif /* SecString_h */
   ```

3. `<Project-Name>-Bridging-Header.h` where `Project-Name` is the name of the project:

   ```c
   #include "SecString.h"
   ```

4. `SecString.swift`:

   ```swift
   import Foundation
   func SecString(_ s:String) -> String {
       return OCSecString(s) as String
   }

   extension OCSecString {
       public convenience init(_ s:String) {
           self.init(string:s)
       }
   }
   ```

That's it, you're ready to go!
