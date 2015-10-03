# RNCryptor

Cross-language AES Encryptor/Decryptor [data format](https://github.com/RNCryptor/RNCryptor-Spec/blob/master/RNCryptor-Spec-v3.md).
 
The primary targets are Swift and Objective-C, but implementations are available in [C](https://github.com/RNCryptor/RNCryptor-C), [C++](https://github.com/RNCryptor/RNCryptor-cpp), [C#](https://github.com/RNCryptor/RNCryptor-cs), [Erlang](https://github.com/RNCryptor/RNCryptor-erlang), [Go](https://github.com/RNCryptor/RNCryptor-go), [Haskell](https://github.com/RNCryptor/rncryptor-hs), [Java](https://github.com/RNCryptor/JNCryptor),
[PHP](https://github.com/RNCryptor/RNCryptor-php), [Python](https://github.com/RNCryptor/RNCryptor-python),
[Javascript](https://github.com/chesstrian/JSCryptor), and [Ruby](https://github.com/RNCryptor/ruby_rncryptor).

The data format includes all the metadata required to securely implement AES encryption, as described in ["Properly encrypting with AES with CommonCrypto,"](http://robnapier.net/aes-commoncrypto) and [*iOS 6 Programming Pushing the Limits*](http://iosptl.com), Chapter 15. Specifically, it includes:

* AES-256 encryption
* CBC mode
* Password stretching with PBKDF2
* Password salting
* Random IV
* Encrypt-then-hash HMAC

## Contents

* [Format Versus Implementation](#format-versus-implementation)
* [Basic Password Usage](#basic-password-usage)
* [Incremental Usage](#incremental-usage)
* [Installation](#installation)
* [Advanced Usage](#advanced-usage)
* [FAQ](#faq)
* [Design Considerations](#design-considerations)
* [License](#license)

## Format Versus Implementation

The RNCryptor data format is cross-platform and there are many implementations. The framework named "RNCryptor" is a specific implementation for Swift and Objective-C. Both have version numbers. The current data format is v3. The current framework implementation (which reads the v3 format) is v4. 

## Basic Password Usage

### Swift

```swift
// Encryption
let data: NSData = ...
let password = "Secret password"
let ciphertext = RNCryptor.encryptData(data, password: password)

// Decryption
do {
    let originalData = try RNCryptor.decryptData(ciphertext, password: password)
    // ...
} catch {
    print(error)
}

```

### Obj-C

``` objc
// Encryption
NSData *data = ...
NSString *password = @"Secret password";
NSData *ciphertext = [RNCryptor encryptData:data password:password];

// Decryption
NSError *error = nil;
NSData *plaintext = [RNCryptor decryptData:ciphertext password:password error:&error];
if (error != nil) {
    NSLog(@"ERROR:", error);
    return
}
// ...
```

## Incremental Usage

RNCryptor suports incremental use, specifically designed to work with `NSURLSession`. This is also useful for cases where the encrypted or decrypted data will not comfortably fit in memory.

To operate in incremental mode, you create an `Encryptor` or `Decryptor`, call `updateWithData()` repeatedly, gathering its results, and then call `finalData()` and gather its result.

### Swift

```swift
//
// Encryption
//
let password = "Secret password"
let encryptor = RNCryptor.Encryptor(password: password)
let ciphertext = NSMutableData()

// ... Each time data comes in, update the encryptor and accumulate some ciphertext ...
ciphertext.appendData(encryptor.updateWithData(data))

// ... When data is done, finish up ...
ciphertext.appendData(encryptor.finalData())

//
// Decryption
//
let password = "Secret password"
let decryptor = RNCryptor.Decryptor(password: password)
let plaintext = NSMutableData()

// ... Each time data comes in, update the decryptor and accumulate some plaintext ...
try plaintext.appendData(decryptor.updateWithData(data))

// ... When data is done, finish up ...
try plaintext.appendData(decryptor.finalData())
```

### Obj-C

``` objc
//
// Encryption
//
NSString *password = @"Secret password";
RNEncryptor *encryptor = [[RNEncryptor alloc] initWithPassword:password];
NSMutableData *ciphertext = [NSMutableData new];

// ... Each time data comes in, update the encryptor and accumulate some ciphertext ...
[ciphertext appendData:[encryptor updateWithData:data]];

// ... When data is done, finish up ...
[ciphertext appendData:[encryptor finalData]];


//
// Decryption
//
RNDecryptor *decryptor = [[RNDecryptor alloc] initWithPassword:password];
NSMutableData *plaintext = [NSMutableData new];

// ... Each time data comes in, update the decryptor and accumulate some plaintext ...
NSError *error = nil;
NSData *partialPlaintext = [decryptor updateWithData:data error:&error];
if (error != nil) { 
    NSLog(@"FAILED DECRYPT: %@", error);
    return;
}
[plaintext appendData:partialPlaintext];

// ... When data is done, finish up ...
NSError *error = nil;
NSData *partialPlaintext = [decryptor finalDataAndReturnError:&error];
if (error != nil) { 
    NSLog(@"FAILED DECRYPT: %@", error);
    return;
}

[ciphertext appendData:partialPlaintext];

```

## Installation

### Requirements

RNCryptor 4 is written in Swift 2, so requires Xcode 7, and can target iOS 7 or greater and OS X 10.9 or later. If you want a pure ObjC implementation that supports older versions of iOS and OS X, see [RNCryptor 3](https://github.com/RNCryptor/RNCryptor/releases/tag/RNCryptor-3.0.1).

### A word about CommonCrypto

CommonCrypto hates Swift. That may be an overstatment. CommonCrypto is...apathetic about Swift to the point of hostility. Apple only needs to do a few things to make CommonCrypto a fine Swift citizen, but as of Xcode 7, those things have not happened, and this makes it difficult to import CommonCrypto into Swift projects.

The most critical thing is that CommonCrypto is not a module, and Swift can't really handle things that aren't modules. The RNCryptor project comes with `CommonCrypto.framework`, which is basically a fake module. Its only function is to tell Swift where the CommonCrypto headers live. It doesn't contain any CommonCrypto code. You don't even need to link it. For maximum robustness across Xcode versions, `CommonCrypto.framework` points to `/usr/include`. The CommonCrypto headers change very rarely, so this shouldn't cause any problem. Hopefully Apple will finally make a module around CommonCrypto and this won't be necessary in the future.

### Installing as a subproject

The easiest way to use RNCryptor is as a subproject without a framework. RNCryptor is just one file, and you can skip all the complexity of managing frameworks this way. It also makes version control very simple if you use submodules, or checkin specific versions of RNCryptor to your repository.

This process works for most targets: iOS and OS X GUI apps, Swift frameworks, and OS X commandline apps. **It is not safe for ObjC frameworks or frameworks that may be imported into ObjC, since it would cause duplicate symbols if some other framework includes RNCryptor.**

* Drag `RNCryptor.xcodeproj` into your project
* Drag `RNCryptor.swift` from `RNCryptor` into your project
* In your target build settings, Build Phases, add `CommonCrypto.framework` as a build dependency (but don't link it).

Built this way, you don't need to (and can't) `import RNCryptor` into your code. RNCryptor will be part of your module.

### Installing without a subproject

If you want to keep things as small and simple as possible, you don't need the full RNCryptor project at all. You just need two things: `RNCryptor.swift` and `CommonCrypto.framework`. You can just copy those into your project.

The same warnings apply as for subprojects: **It is not safe for ObjC frameworks or frameworks that may be imported into ObjC, since it would cause duplicate symbols if some other framework included RNCryptor.**

* Copy or link `RNCryptor/RNCryptor.swift` into your project.
* Copy or link `CommonCrypto.framework` into your project.
* Go to your target build settings, Build Phases, and delete `CommonCrypto.framework` from "Link Binary with Libraries." You don't actually want to link it.

Built this way, you don't need to (and can't) `import RNCryptor` into your code. RNCryptor will be part of your module.

### [Carthage](https://github.com/Carthage/Carthage)

    github "RNCryptor/RNCryptor" "~> 4.0"

Don't forget to copy and link `RNCryptor.framework` and at least copy `CommonCrypto.framwork`. (You can link the Carthage version of `CommonCrypto.framework`. It doesn't matter.)

This approach will not work for OS X commandline apps.

### [CocoaPods](https://cocoapods.org)

    pod 'RNCryptor', '~> 4.0'

This approach will not work for OS X commandline apps.

## Advanced Usage

### Version-Specific Cryptors

The default `RNCryptor.Encryptor` is the "current" version of the data format (currently v3). If you're interoperating with other implementations, you may need to choose a specific format for compatibility.

To create a version-locked cryptor, use `RNCryptor.EncryptorV3` and `RNCryptor.DecryptorV3`.

Remember: the version specified here is the *format* version, not the implementation version. The v4 RNCryptor framework reads and writes the v3 RNCryptor data format.

### Key-Based Encryption

*You need a little expertise to use key-based encryption correctly, and it is very easy to make insecure systems that look secure. The most important rule is that keys must be random across all their bytes. If you're not comfortable with basic cryptographic concepts like AES-CBC, IV, and HMAC, you probably should avoid using key-based encryption.*

Cryptography works with keys, which are random byte sequences of a specific length. The RNCryptor v3 format uses two 256-bit (32-byte) keys to perform encryption and authentication.

Passwords are not "random byte sequences of a specific length." They're not random at all, and they can be a wide variety of lengths, very seldom exactly 32. RNCryptor defines a specific and secure way to convert passwords into keys, and that is one of it's primary features.

Occasionally there are reasons to work directly with random keys. Converting a password into a key is intentionally slow (tens of milliseconds). Password-encrypted messages are also a 16 bytes longer than key-encrypted messages. If your system encrypts and decrypts many short messages, this can be a significant performance impact, particularly on a server.

RNCryptor supports direct key-based encryption and decryption. The size and number of keys may change between format versions, so key-based cryptors are [version-specific](#version-specific-cryptors).

In order to be secure, the keys must be a random sequence of bytes. If you're starting with a string of any kind, you are almost certainly doing this wrong.

```swift
let encryptor = RNCryptor.EncryptorV3(encryptionKey: encryptKey, hmacKey: hmacKey)
let decryptor = RNCryptor.DecryptorV3(encryptionKey: encryptKey, hmacKey: hmacKey)
```

```objc
RNEncryptor *encryptor = [[[RNEncryptorV3 alloc] initWithEncryptionKey:encryptionKey hmacKey:hmacKey];
RNDecryptor *decryptor = [[[RNDecryptorV3 alloc] initWithEncryptionKey:encryptionKey hmacKey:hmacKey];
```

## FAQ

### Can I use RNCryptor to read and write my non-RNCryptor data format?

No. RNCryptor implements a specific data format. It is not a general-purpose encryption library. If you have created your own data format, you will need to write specific code to deal with whatever you created. Please make sure the data format you've invented is secure. (This is much harder than it sounds.)

If you're using the OpenSSL encryption format, see [RNOpenSSLCryptor](https://github.com/rnapier/RNOpenSSLCryptor).

### Can I change the parameters used (algorithm, iterations, etc)?

No. See previous question. The v4 format will permit some control over PBKDF2 iterations, but the only thing configurable in the v3 format is whether a password or key is used. This keeps RNCryptor implementations dramatically simpler and interoperable.

### How do I encrypt/decrypt a string?

AES encrypts bytes. It does not encrypt characters, letters, words, pictures, videos, cats, or ennui. It encrypts bytes. You need to convert other data (such as strings) to and from bytes in a consistent way. There are several ways to do that. Some of the most popular are UTF-8 encoding, Base-64 encoding, and Hex encoding. There are many other options. There is no good way for RNCryptor to guess which encoding you want, so it doesn't try. It accepts and returns bytes in the form of `NSData`.

To convert strings to data as UTF-8, use `dataUsingEncoding()` and `init(data:encoding:)`. To convert strings to data as Base-64, use `init(base64EncodedString:options:)` and `base64EncodedStringWithOptions()`.

### Does RNCryptor support random access decryption?

The usual use case for this is encrypting media files like video. RNCryptor uses CBC encryption, which prevents easy random-access. While other modes are better for random-access (CTR for instance), they are more complicated to implement correctly and CommonCrypto doesn't directly support using them for random access anyway.

That said, it would be fairly easy to build a wrapper around RNCryptor that allowed random-access to blocks of some fixed size (say 64k), and that could work well for video with modest overhead (see [inferno](http://securitydriven.net/inferno/) for a very similar format in C#). Such a format would be fairly easy to port to other platforms that already support RNCryptor.

If there is interest, I may eventually build this as a separate framework. If you have a pressing need sooner than "eventually," please contact me about rates.

## Design Considerations

`RNCryptor` has several design goals, in order of importance:

### Easy to use correctly for most common use cases

The most critical concern is that it be easy for non-experts to use `RNCryptor` correctly. A framework that is more secure, but requires a steep learning curve on the developer will either be not used, or used incorrectly. Whenever possible, a single line of code should "do the right thing" for the most common cases.

This also requires that it fail correctly and provide good errors.

### Reliance on CommonCryptor functionality

`RNCryptor` has very little "security" code. It relies as much as possible on the OS-provided CommonCryptor. If a feature does not exist in CommonCryptor, then it generally will not be provided in `RNCryptor`.

### Best practice security

Wherever possible within the above constraints, the best available algorithms
are applied. This means AES-256, HMAC+SHA256, and PBKDF2. (Note that several of these decisions were reasonable for v3, but may change for v4.)

* AES-256. While Bruce Schneier has made some interesting recommendations
regarding moving to AES-128 due to certain attacks on AES-256, my current
thinking is in line with [Colin
Percival](http://www.daemonology.net/blog/2009-07-31-thoughts-on-AES.html).
PBKDF2 output is effectively random, which should negate related-keys attacks
against the kinds of use cases we're interested in.

* AES-CBC mode. This was a somewhat complex decision, but the ubiquity of CBC
outweighs other considerations here. There are no major problems with CBC mode,
and nonce-based modes like CTR have other trade-offs. See ["Mode changes for
RNCryptor"](http://robnapier.net/mode-rncryptor) for more details on this
decision.

* Encrypt-then-MAC. If there were a good authenticated AES mode on iOS (GCM for
instance), I would probably use that for its simplicity. Colin Percival makes
[good arguments for hand-coding an encrypt-than-
MAC](http://www.daemonology.net/blog/2009-06-24-encrypt-then-mac.html) rather
than using an authenticated AES mode, but in RNCryptor mananging the HMAC
actually adds quite a bit of complexity. I'd rather the complexity at a more
broadly peer-reviewed layer like CommonCryptor than at the RNCryptor layer. But
this isn't an option, so I fall back to my own Encrypt-than-MAC.

* HMAC+SHA256. No surprises here.

* PBKDF2. While bcrypt and scrypt may be more secure than PBKDF2, CommonCryptor
only supports PBKDF2. [NIST also continues to recommend
PBKDF2](http://security.stackexchange.com/questions/4781/do-any-security-experts-recommend-bcrypt-for-password-storage). We use 10k rounds of PBKDF2
which represents about 80ms on an iPhone 4.

### Code simplicity

RNCryptor endeavors to be implemented as simply as possible, avoiding tricky code. It is designed to be easy to read and code review.

### Performance

Performance is a goal, but not the most important goal. The code must be secure
and easy to use. Within that, it is as fast and memory-efficient as possible.

### Portability

Without sacrificing other goals, it is preferable to read the output format of
`RNCryptor` on other platforms.

## License

Except where otherwise indicated in the source code, this code is licensed under
the MIT License:

>Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

>The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

>THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NON-INFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE. ```
