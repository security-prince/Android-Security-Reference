#KeyStore

> KeyStore is responsible for maintaining cryptographic keys and their owners.
 
and 
 
> Use the Android Keystore provider to let an individual app store its own credentials that only the app itself can access. This provides a way for apps to manage credentials that are usable only by itself while providing the same security benefits that the KeyChain API provides for system-wide credentials. This method requires no user interaction to select the credentials.

##Why should I use it?

If you need to utilise and persist a cryptographic key in your application you either need to:

1. Request some input from the user (like a password) and either use this directly to derive a key or use it to encrypt a key which can then be persisted to disk
2. Store the key securely so it cant 'easily' be snooped

Some implementations may store a key inside shared prefs or raw on the filesystem, or worse, in the source code, which is a vulnerable approach to key management. 

KeyStore allows a way to store a key in hardware (if available) or in software (as an encrypted blob based off the device lock key) making it much harder to lift keys.

This is especially handy for any kind of challenge / response auth or other process where a compromise of the key itself would cause problems.


##Related API

- [`KeyStore`](http://developer.android.com/reference/java/security/KeyStore.html)
  - KeyStore is responsible for maintaining cryptographic keys and their owners.
- Used with   
  - [`Cipher`](http://developer.android.com/reference/javax/crypto/Cipher.html#init(int, java.security.Key))
    - `Cipher.init(...)` takes a [`Key`](http://developer.android.com/reference/java/security/Key.html) which can represent a key that lives in the `KeyStore`
    - From **J-4.3-18** this can be the output of `KeyPair.getPrivate()`, where `KeyPair` is obtained from `KeyStore.getEntry(...)` e.g. RSA
    - From **M-6-23** this can be the output of `KeyStore.getKey(..)` if the `Key` for the passes `alias` is a `SecretKey` e.g. AES
    - `KeyStore` supported `Cipher` list is found on the [Android Keystore System](http://developer.android.com/training/articles/keystore.html) page
  - [`Key`](http://developer.android.com/reference/java/security/Key.html)
	    - Key is the common interface for all keys.  
  - Symmetric   
    - [`SecretKey`](http://developer.android.com/reference/javax/crypto/SecretKey.html) 
      - `KeyStore.getEntry(alias)` should return a `KeyStore.SecretKeyEntry` if `alias` represents an AES key
      - `KeyStore.getKey(alias)` should return a `SecretKey` if `alias` represents an AES key
    - [`KeyGenerator`](https://developer.android.com/reference/javax/crypto/KeyGenerator.html)
      - This class provides the public API for generating symmetric cryptographic keys. 
  - Asymmetric
	  - [`KeyPair`](http://developer.android.com/reference/java/security/KeyPair.html)
	    - KeyPair is a container for a public key and a private key. 
	    - `KeyStore.getEntry(..)` should return a `KeyStore.PrivateKeyEntry`
	  - [`KeyPairGenerator`](http://developer.android.com/reference/java/security/KeyPairGenerator.html)
	    - KeyPairGenerator is an engine class which is capable of generating a private key and its related public key utilizing the algorithm it was initialized with. 
	    - `KeyPairGenerator.getInstance(<cipherName>, "AndroidKeyStore");` for the key to be generated in the system keystore.
	  - [`KeyPairGeneratorSpec`](http://developer.android.com/reference/android/security/KeyPairGeneratorSpec.html)
	    - This provides the required parameters needed for initializing the KeyPairGenerator that works with Android `KeyStore`. The spec determines authorized uses of the key, such as whether user authentication is required for using the key, what operations are authorized, with what parameters, and the key's validity start and end dates
	    - **This class was deprecated in API level 23. Use `KeyGenParameterSpec` instead.**
  - Both
    - [`KeyGenParameterSpec`](https://developer.android.com/reference/android/security/keystore/KeyGenParameterSpec.html)
      - `AlgorithmParameterSpec` for initializing a `KeyPairGenerator` or a `KeyGenerator` of the Android Keystore system. 
      - **Since M-6-23**
      - More control over what the key can be used for and when
      - Supports authentication-required-before-use (lock screen | finger)

##Version changes

- **D-1.6-4**
  - Keystore implemented as a native keystore daemon that used a local socket as its IPC interface
- **J-4.3-18**
  - [Credential storage enhancements in Android 4.3](http://nelenkov.blogspot.co.uk/2013/08/credential-storage-enhancements-android-43.html)
  - can store public / private keys via RSA `Cipher` support
  - `Signiture` support for RSA varients
  - can have multiple device users
    - before 4.3 only PRIMARY user could use
  - CANT store AES keys
    - but could encypt the AES key with created public / private key and in keystore
    - can store directly via hidden apis - see link above
    - Can with [`Vault`](https://android.googlesource.com/platform/development/+/master/samples/Vault/src/com/example/android/vault/SecretKeyWrapper.java) wrapper class
      - check out [g+ post](https://plus.google.com/+JeffSharkey/posts/9BmGb3xbPcA)
  - [Android 4.3 release notes](http://developer.android.com/about/versions/android-4.3.html#Security)
  - deamon retired and replaced with binder interface
- **K-4.4-19**
  - Support for elliptic curve
  - `Signiture` support for ECDSA varients
- **M-6-23**
  - First time for AES `Cipher` support (See ['Supported Algorithms'](http://developer.android.com/training/articles/keystore.html#SupportedAlgorithms)) 
  - `KeyGenerator` support for AES & HMAC
  - `KeyFactory` & `KeyPairGenerator` support for EC & RSA
  - `Mac` Hmac support
  - `SecretKeyFactory` support for AES & Hmac 
  - `Signature` increased support 
  - "Keys which do not require encryption at rest will no longer be deleted when secure lock screen is disabled or reset (for example, by the user or a Device Administrator). Keys which require encryption at rest will be deleted during these events.
  - [`KeyProtection`](http://developer.android.com/reference/android/security/keystore/KeyProtection.html)
    - Specification of how a key or key pair is secured when imported into the Android Keystore system.
  - `KeyStore` keys can require fingerprint auth before use 
    - "User authentication authorizes a specific cryptographic operation associated with one key. In this mode, each operation involving such a key must be individually authorized by the user". See `api/finger` for more.
- **N-7?-24?**
  - Key Attestion
    - Can prove to 3rd partys that a hardware keystore exists with certain keys by signing a representation using a factory supplied key 
    - KeyStore now returns a chain for a key alias which can be used to verify that the device has passed CTS testing
  - Hardware KeyStore manditory [IO link](https://youtu.be/XZzLjllizYs?t=571) 

##Authenticating Key Use

- Used in 3 User related modes
	- Not encrypted (default)
	  - If using the `KeyStore` without any additional settings just unlocking the device should unlock the `KeyStore` 
	- [`.setEncryptionRequired()`](http://developer.android.com/reference/android/security/KeyPairGeneratorSpec.Builder.html#setEncryptionRequired())  
	  - Requires a lock screen is set 
	- `KeyGenParameterSpec.Builder.setUserAuthenticationRequired(true)` 
	   - User has to autheticate each use of the key (see below for more)	   
	   - Optional `.setUserAuthenticationValidityDurationSeconds(AUTHENTICATION_DURATION_SECONDS)`
	     - Previous key validations are valid for this amount of time 
- OS access auth
  - Key access is tied to the apps UID
  - If rooted any user/app can in theory assume any UID
  - Access marshalled by keystore deamon on older apis and binder server on new ones 
- Crypto uses auth
  - authorized key algorithm, operations or purposes (encrypt, decrypt, sign, verify), padding schemes, block modes, digests with which the key can be used;   
 
###More on `.setUserAuthenticationRequired(true)` 

If a key has been created with `.setUserAuthenticationRequired(true)` then the user has to either authenticate via 

- [`KeyGuardManager.createConfirmDeviceCredentialIntent(...)`](https://developer.android.com/reference/android/app/KeyguardManager.html#createConfirmDeviceCredentialIntent(java.lang.CharSequence, java.lang.CharSequence)) to promt the user to enter pin, pattern or password. 
  - [ConfirmCredential](http://developer.android.com/samples/ConfirmCredential/index.html)
    - Uses `.setUserAuthenticationValidityDurationSeconds(AUTHENTICATION_DURATION_SECONDS)`  
    - This sample demonstrates how you can use device credentials (PIN, Pattern, Password) in your app to authenticate the user before they are trying to complete some actions
    - Will try to use key as part of crypto op after auth
- Fingerprint
  - Check [fingerprint.md](fingerprint.md) for more. Essentially needs to perform `KeyStore` op in fingerprint api auth callback.

##Losing Keys and how to handle

See [Android Security: The Forgetful Keystore](http://doridori.github.io/android-security-the-forgetful-keystore/) for losing Keys due to lock screen changes.

You can also lose Keys that require fingerprint auth when a new finger is enrolled. See [fingerprint.md](/api/fingerprint.md)

##Locking of Keystore

I have not encountered this myself but there are anecdontal reports of keystores becoming locked even when `setEncryptionRequired` == false. See [this SO post](http://stackoverflow.com/a/25790891/236743).

##Hardware vs Software & CTS

Some devices have hardware backed keystore storage, which was introduced in 4.3 but not manditory (Nexus devices since Nexus 4 have had this afaik). If a [CTS](http://static.googleusercontent.com/media/source.android.com/en//compatibility/android-cdd.pdf) compatable fingerprint functionality is available then

> MUST have a hardware-backed keystore implementation, and perform the fingerprint matching in a Trusted Execution Environment (TEE) or on a chip with a secure channel to the TEE.

_A [TEE](https://en.wikipedia.org/wiki/Trusted_execution_environment) (or [TPM](https://en.wikipedia.org/wiki/Trusted_Platform_Module)) can change a `KeyStore` attack from **offline** (which ould be against a software KeyStores data) to **online** (running on the device issuing commands to the hardware). This further prevents extraction of the private key and potentially throttling of access attempts (and therefore potential exploitation). See Androids [Trusty TEE](https://source.android.com/security/trusty/index.html)_

As of M-6-23 hardware backed keystore is not a requirement, but is [now manditory in N](https://youtu.be/XZzLjllizYs?t=571).

> Note that while the above TEE-related requirements are stated as STRONGLY RECOMMENDED, the
Compatibility Definition for the next API version is planned to changed these to REQIUIRED. If a
device implementation is already launched on an earlier Android version and has not implemented a
trusted operating system on the secure hardware, such a device might not be able to meet the
requirements through a system software update and thus is STRONGLY RECOMMENDED to
implement a TEE.

_Taken from [6.0 CTS](http://static.googleusercontent.com/media/source.android.com/en//compatibility/android-cdd.pdf) doc section 9.11. Keys and Credentials_

The presence of hardware backed key storage can be checked via the [`KeyChain.isBoundKeyAlgorithm`](http://developer.android.com/reference/android/security/KeyChain.html#isBoundKeyAlgorithm(java.lang.String)) method and for M-6-23+ [`KeyInfo.isInsideSecureHardware ()`](http://developer.android.com/reference/android/security/keystore/KeyInfo.html#isInsideSecureHardware()). See below

> Android also now supports hardware-backed storage for your KeyChain credentials, providing more security by making the keys unavailable for extraction. That is, once keys are in a hardware-backed key store (Secure Element, TPM, or TrustZone), they can be used for cryptographic operations but the private key material cannot be exported. Even the OS kernel cannot access this key material. While not all Android-powered devices support storage on hardware, you can check at runtime if hardware-backed storage is available by calling KeyChain.IsBoundKeyAlgorithm().

This means on a hardware backed device even if rooted the private keys cannot be extracted! They can still be used locally however (by an attacker).

> Key material may be bound to the secure hardware (e.g., Trusted Execution Environment (TEE), Secure Element (SE)) of the Android device. When this feature is enabled for a key, its key material is never exposed outside of secure hardware. If the Android OS is compromised or an attacker can read the device's internal storage, the attacker may be able to use any app's Android Keystore keys on the Android device, but not extract them from the device. This feature is enabled only if the device's secure hardware supports the particular combination of key algorithm, block modes, padding schemes, and digests with which the key is authorized to be used. To check whether the feature is enabled for a key, obtain a KeyInfo for the key and inspect the return value of KeyInfo.isInsideSecurityHardware().

and 

> If the Android OS is compromised or an attacker can read the device's internal storage, the attacker may be able to use any app's Android Keystore keys on the Android device, but not extract them from the device

The software blob can however be read on rooted devices. See [nelenkov/keystore-decryptor](https://github.com/nelenkov/keystore-decryptor).

##CAs

CAs are also stored in the `KeyStore`. When added a custom CA device should prompt user to enter device credentials.


##Links

- [`KeyStore`](http://developer.android.com/reference/java/security/KeyStore.html) class
- [developer.android.com] [Android Keystore System](http://developer.android.com/training/articles/keystore.html)
  - Shows list of Ciphers (& more) supported by the `KeyStore` 
- [Nikolay Elenkov] [Keystore redesign in Android M](https://nelenkov.blogspot.co.uk/2015/06/keystore-redesign-in-android-m.html)
- Bugs / Vulns
  - [Android keystore key leakage between security domains](http://jbp.io/2014/04/07/android-keystore-leak/)
    - Use `setEncryptionRequired()` to mitigate 


