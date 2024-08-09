[![](https://jitpack.io/v/yongjhih/pepk.svg)](https://jitpack.io/#yongjhih/pepk)
[![javadoc](https://img.shields.io/github/tag/yongjhih/pepk.svg?label=javadoc)](https://jitpack.io/com/github/yongjhih/pepk/-SNAPSHOT/javadoc/)
[![LICENSE](https://img.shields.io/packagist/l/yongjhih/pepk.svg?style=flat)](LICENSE)

# PEPK - Play Encrypt Private Key

pepk (Play Encrypt Private Key) is a tool for exporting private keys from a
Java Keystore and encrypting them for transfer to Google Play as part of
enrolling in App Signing by Google Play.

Use this code (the code performs EC-P256+AES-GCM hybrid encryption) with the hex encoded public key (a 4-byte identity followed by a 64-byte P256 point) to create your own tool exporting in a zip file the encrypted private key and the public certificate in PEM format.

## About

This repository is totally imported from Google [pepk-src.jar](https://www.gstatic.com/play-apps-publisher-rapid/signing-tool/prod/pepk-src.jar),
just make it capable of modifying and building with gradle.

## Usage of command line

Use the command below to run the tool, which will export and encrypt your private key and its public certificate. Ensure that you replace the arguments. Then enter your store and key passwords when prompted.

Using [installed](#installation) pepk.jar

```sh
java -jar pepk.jar --keystore=foo.keystore --alias=foo --output=output.zip --encryptionkey=xxx --include-cert
```

or using docker:

```sh
docker run --rm -it -v $(pwd):$(pwd) -w $(pwd) yongjhih/pepk --keystore=foo.keystore --alias=foo --output=output.zip --encryptionkey=xxx --include-cert
```

or using [docker-pepk](docker-pepk) into `~/bin/pepk`:

```sh
curl -L https://raw.githubusercontent.com/yongjhih/pepk/master/docker-pepk -o ~/bin/pepk && chmod a+x ~/bin/pepk
pepk --keystore=foo.keystore --alias=foo --output=output.zip --encryptionkey=xxx --include-cert
```

You will see the output.zip contains

```
├── certificate.pem
└── encryptedPrivateKey
```

## Usage of library

```kotlin
val keyStore = KeyStore.getInstance("jks").apply {
    File("signing.jks").inputStream().use { input -> load(input, "storepass".toCharArray()) }
}
val key = keyStore.getKey("signing", "keypass".toCharArray()) as PrivateKey

fun String.decodeHex(): ByteArray = this.chunked(2).map { it.toInt(16).toByte() }.toByteArray()

val encryptedPrivateKey = KeymaestroHybridEncrypter("eb10fe8f7c7c9df715022017b00c6471f8ba8170b13049a11e6c09ffe3056a104a3bbe4ac5a955f4ba4fe93fc8cef27558a3eb9d2a529a2092761fb833b656cd48b9de6a".decodeHex())
                            .encrypt(ExportEncryptedPrivateKeyTool.privateKeyToPem(key))

println(encryptedPrivateKey)
```

## Development

```sh
./gradlew run --args='--keystore=foo.keystore --alias=foo --output=output.zip --encryptionkey=xxx --include-cert'
```

or

```sh
./gradlew shadowJar
java -jar build/libs/pepk.jar --keystore=foo.keystore --alias=foo --output=output.zip --encryptionkey=xxx --include-cert
```

## Deployment

```sh
./gradlew shadowJar
```

Upload build/libs/pepk.jar into [releases](https://github.com/yongjhih/pepk/releases)

## Installation of command line tool

If you dont use docker, you have to download pepk.jar from [releases](https://github.com/yongjhih/pepk/releases)

```sh
java -jar pepk.jar ARGS
```

## Installation of as library

```gradle
repositories {
    maven { url 'https://jitpack.io' }
}

dependencies {
    implementation 'com.github.yongjhih:pepk:-SNAPSHOT'
}
```

## Bonus - android pepk gradle

Auto generate encrytpedPrivateKey file by applying gradle script:


```gradle
apply from: 'pepk.gradle'

android {
    release {
        signingConfig {
            storeFile file("${projectDir}/signing-release.jks")
            // ...
        }
    }
}
```

Add pepk.gradle and modify the encryption key:

```gradle
buildscript {
    repositories {
        jcenter()
        maven { url 'https://jitpack.io' }
    }
    dependencies {
        classpath 'com.github.yongjhih:pepk:-SNAPSHOT'
    }
}

import com.google.security.keymaster.lite.KeymaestroHybridEncrypter
import com.google.wireless.android.vending.developer.signing.tools.extern.export.ExportEncryptedPrivateKeyTool
import java.security.KeyStore
import java.security.PrivateKey

afterEvaluate {
    android.applicationVariants.all { variant ->
        if (variant.signingConfig && file(variant.signingConfig.storeFile).exists() && !file("${variant.signingConfig.storeFile}.encryptedPrivateKey").exists()) {
            def keyStore = KeyStore.getInstance("jks")
            variant.signingConfig.storeFile.withInputStream { stream ->
                keyStore.load(stream, variant.signingConfig.storePassword.toCharArray())
            }
            def key = keyStore.getKey(variant.signingConfig.keyAlias, variant.signingConfig.keyPassword.toCharArray())
            def encryptedPrivateKey = new KeymaestroHybridEncrypter("eb10fe8f7c7c9df715022017b00c6471f8ba8170b13049a11e6c09ffe3056a104a3bbe4ac5a955f4ba4fe93fc8cef27558a3eb9d2a529a2092761fb833b656cd48b9de6a".decodeHex())
                .encrypt(ExportEncryptedPrivateKeyTool.privateKeyToPem(key))
            file("${variant.signingConfig.storeFile}.encryptedPrivateKey").withOutputStream {
                it.write encryptedPrivateKey
            }
        }
    }
}
```

You will see `${projectDir}/signing-release.jks.encryptedPrivateKey` file has been generated.

## Changelogs

Latest known pepk.jar/build-data.properties:

```
build.target=blaze-out/k8-opt/bin/third_party/java/pepk/ExportEncryptedPrivateKeyTool_deploy.jar
main.class=com.google.wireless.android.vending.developer.signing.tools.extern.export.ExportEncryptedPrivateKeyTool
build.gplatform=gcc-4.X.Y-crosstool-v18-llvm-grtev4-k8
build.depot.path=//depot/branches/play-apps-publisher-app-signing-encryption-tool_release_branch/229150960.1/google3
build.client_mint_status=1
build.client=build-secure-info\:source-uri
build.verifiable=1
build.location=play-apps-publisher-releaser@iojf11.prod.google.com\:/google/src/files/229167424/depot/branches/play-apps-publisher-app-signing-encryption-tool_release_branch/229150960.1/OVERLAY_READONLY/google3
build.tool=Blaze, release blaze-2018.12.11-3 (mainline @224801075)
build.label=play-apps-publisher-app-signing-encryption-tool_20190114_RC00
build.timestamp.as.int=1547473306
build.versionmap=map 0 { // }
build.changelist.as.int=229167424
build.baseline.changelist.as.int=229150960
build.timestamp=1547473306
build.build_id=1221793a-c127-4962-815a-1f6b61ad2cca
build.changelist=229167424
build.time=Mon Jan 14 05\:41\:46 2019 (1547473306)
build.citc.snapshot=-1
```

## References

* https://developer.android.com/studio/publish/app-signing
* Original [pepk.jar](https://www.gstatic.com/play-apps-publisher-rapid/signing-tool/prod/pepk.jar)
* Original [pepk-src.jar](https://www.gstatic.com/play-apps-publisher-rapid/signing-tool/prod/pepk-src.jar)

## License

Apache 2.0 - Google Inc, 2017
