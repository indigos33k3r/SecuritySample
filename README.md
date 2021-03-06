# SecuritySample
Hiding encrypted secret API keys in C/C++ code and decrypting them via JNI.

>Native code is harder to decompile than Java code. That's what we write secret keys in C/C++ code. 
>To be safer, you can encrypt those secret keys before you fill in them. So you have to decrypt them to use.

Using SafetyNet Attestation APIs.

>SafetyNet is a nifty solution in the following scenarios:      
>1. I'm not sure if the app which is connecting to my server is that app I published.
>2. Can I trust this Android API?       
>3. Is this a real, compatible device?      
>4. Whether my application is running on a rooted device or not.

Well, safetyNet APIs are used to evaluate if the environment where your app runs is safe and compatible with the Android API or not. To check the integrity, compatibility and signature of your app by calling Attestation APIs. Let your server decide to continue or to stop connecting to that untrusted device immediately.

# Features

## 1. Get encrypted data from native code through NDK

- Hiding Secret keys in C/C++ code.
- Using [RSAHelper] to encrypt your secret keys (For example, authorization key, public key, iv parmeter of DES algorithm or something). Paste those encrypted strings to this project. Then, fill in the parameters generated by [RSAHelper] in this project for decryption to decrypt the messages.

## 2. Evaluate the security and compatibility of the Android environments in which your apps run
- Call SafetyNet Attestation APIs.

# Instruction of JNI, encryption and decryption

In the begining, you might need to create a keystore.properties file to keep some information you need.

```
storeFile=/Users/workspace/Keystores/xxx.jks
storePassword=xxxx
keyAlias=xxxx
keyPassword=xxxx
```

## Step1. Generate a pair of RSA keys and encrypt your messages.
Run [RSAHelper] to get encrypted messages, using RSA modulus and exponent for decryption.

## Step2. Fill in MODULUS and EXPONENT
Hide RSA parameters in [Config.cpp]

```C
JNIEXPORT jobjectArray JNICALL
Java_com_catherine_securitysample_JNIHelper_getKeyParams(JNIEnv *env, jobject instance) {
    jobjectArray valueArray = (jobjectArray) env->NewObjectArray(2, env->FindClass("java/lang/String"), 0);
    const char *hash[2];
    //MODULUS
    hash[0] = "Fill in the modulus created by RSAHelper";
    //EXPONENT
    hash[1] = "Fill in the exponent created by RSAHelper";
    for (int i = 0; i < 2; i++) {
        jstring value = env->NewStringUTF(hash[i]);
        env->SetObjectArrayElement(valueArray, i, value);
    }
    return valueArray;
}
```

## Step3. Add the decryption method to your project
In [JNIHelper],

```Java
/**
 * Decrypt messages by RSA algorithm<br>
 *
 * @param message
 * @return Original message
 * @throws NoSuchAlgorithmException
 * @throws NoSuchPaddingException
 * @throws InvalidKeyException
 * @throws IllegalBlockSizeException
 * @throws BadPaddingException
 * @throws UnsupportedEncodingException
 * @throws InvalidAlgorithmParameterException
 * @throws InvalidKeySpecException
 * @throws ClassNotFoundException
 */
public String decryptRSA(String message) throws NoSuchAlgorithmException, NoSuchPaddingException,
        InvalidKeyException, IllegalBlockSizeException, BadPaddingException, UnsupportedEncodingException,
        InvalidAlgorithmParameterException, ClassNotFoundException, InvalidKeySpecException {
    Cipher c2 = Cipher.getInstance(Algorithm.rules.get("RSA")); // 创建一个Cipher对象，注意这里用的算法需要和Key的算法匹配

    BigInteger m = new BigInteger(Base64.decode(getKeyParams()[0].getBytes(), Base64.DEFAULT));
    BigInteger e = new BigInteger(Base64.decode(getKeyParams()[1].getBytes(), Base64.DEFAULT));
    c2.init(Cipher.DECRYPT_MODE, convertStringToPublicKey(m, e)); // 设置Cipher为解密工作模式，需要把Key传进去
    byte[] decryptedData = c2.doFinal(Base64.decode(message.getBytes(), Base64.DEFAULT));
    return new String(decryptedData, Algorithm.CHARSET);
}

/**
 * You can component a publicKey by a specific pair of values - modulus and
 * exponent.
 *
 * @param modulus  When you generate a new RSA KeyPair, you'd get a PrivateKey, a
 *                 modulus and an exponent.
 * @param exponent When you generate a new RSA KeyPair, you'd get a PrivateKey, a
 *                 modulus and an exponent.
 * @throws ClassNotFoundException
 * @throws NoSuchAlgorithmException
 * @throws InvalidKeySpecException
 */
private Key convertStringToPublicKey(BigInteger modulus, BigInteger exponent)
        throws ClassNotFoundException, NoSuchAlgorithmException, InvalidKeySpecException {
    byte[] modulusByteArry = modulus.toByteArray();
    byte[] exponentByteArry = exponent.toByteArray();

    // 由接收到的参数构造RSAPublicKeySpec对象
    RSAPublicKeySpec rsaPublicKeySpec = new RSAPublicKeySpec(new BigInteger(modulusByteArry),
            new BigInteger(exponentByteArry));
    // 根据RSAPublicKeySpec对象获取公钥对象
    KeyFactory kFactory = KeyFactory.getInstance(Algorithm.KEYPAIR_ALGORITHM);
    PublicKey publicKey = kFactory.generatePublic(rsaPublicKeySpec);
    return publicKey;
}
```

## Step4. Create C/C++ files
- There are two ways to use JNI -- CmakeLists.txt and Android.mk, I used Android.mk here.
- Create jni folder in main/ .Then add [Android.mk], [Application.mk] and C/C++ files([Config.cpp]).

![JNI 1][NDK1]

- In build.gradle:

```gradle
externalNativeBuild {
    ndkBuild {
        path 'src/main/jni/Android.mk'
    }
}
```

- In [JNIHelper],

```Java
static {
    //relate to LOCAL_MODULE in Android.mk
    System.loadLibrary("keys");
}
/**
 * A native method that is implemented by the 'native-lib' native library,
 * which is packaged with this application.
 */
public native String[] getAuthChain(String key);

/**
 * A native method that is implemented by the 'native-lib' native library,
 * which is packaged with this application.
 */
public native String[] getKeyParams();
```

## Step5. Run your app

```Java
    private final static String TAG = "MainActivity";

@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        StringBuilder sb = new StringBuilder();
    try {
        // Example of a call to a native method
        TextView tv = (TextView) findViewById(R.id.sample_text);

        String[] authChain = getAuthChain("LOGIN");
        sb.append("Decrypted secret keys\n[ ");
        for (int i = 0; i < authChain.length; i++) {
            sb.append(decryptRSA(authChain[i]));
            sb.append(" ");
        }
        sb.append("]\n");

        String[] authChain2 = getAuthChain("OTHER");
        sb.append("secret keys\n[ ");
        for (int i = 0; i < authChain.length; i++) {
            sb.append(authChain2[i]);
            sb.append(" ");
        }
        sb.append("]");
        Log.d(TAG, sb.toString());
        tv.setText(sb.toString());
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

# Instruction of SafetyNet Attestation APIs

In the begining, you might need to create a keystore.properties file to keep some information you need.

```
storeFile=/Users//Keystores/xxx.jks
storePassword=xxxx
keyAlias=xxxx
keyPassword=xxxx
```

**Things you must know before you start developing.**
1. Use SafetyNetApi the deprecated class or you'd probably get 403 error by calling SafetyNet.getClient(context)
2. JWS (JSON Web Token) contains header, payload and signature, your environment information is refer to the payload.
3. There are two APIs you might need - SafetyNet API and Android device verification API. You get your device and app information with SafetyNet API, and check whether the information is truthful with another. Then let your server decide the next step (like shutting down the app or something).
4. Attestation should not run on your UI thread, you can use HandlerThread to deal with this situation.

>JWS Header : A string representing a JSON object that describes the digital signature or MAC operation applied to create the JWS Signature value.
>JWS Payload : The bytes to be secured -- a.k.a., the message. The payload can contain an arbitrary sequence of bytes.
>JWS Signature : A byte array containing the cryptographic material that secures the JWS Header and the JWS Payload.
> Learn more about JWS : https://tools.ietf.org/html/rfc7515

## Step1. Generate an API key from google developers console (optional)
- **You can skip this step if you don't verify your attestation response from google APIs (I feel like this step is kind of like https validation. It probabily means man-in-the-middle attacks are allowed if you do not check the response.). Of course you can also validate the SSL certificate chain by yourself. Google highly recommends you to check your JWS statement.**
- **What "Android Device Verification API" dose is only checking JWS certificates and signatures. Its response (JSON payload) has nothing to do with the Android environments in which your app run.**
- Get your API key here: https://console.developers.google.com/, and don't forget to add and enable "Android Device Verification API".
- Make sure the API key you post to "Android Device Verification API" is unrestricted.
- There is a daily quota restriction of connecting "Android Device Verification API".

- In gradle.porpeties, add your google API key
```
safetynet_api_key = XXXXXXXXX
```

- In build.gradle
```gradle
android {
  defaultConfig {
          buildConfigField("String", "API_KEY", "\"${safetynet_api_key}\"")
      }
}

```

- DO NOT add any safetyNet meta-data in your manifest
```xml
<!--<meta-data-->
    <!--android:name="com.google.android.safetynet.ATTEST_API_KEY"-->
    <!--android:value="${safetynet_api_key}" />-->
```


## Step2. Build GoogleApiClient and call SafetyNet APIs

In MyApplication,
```java
public class MyApplication extends Application {
    public HandlerThread safetyNetLooper;
    public static MyApplication INSTANCE;

    @Override
    public void onCreate() {
        INSTANCE = this;
        safetyNetLooper = new HandlerThread("SafetyNet task");
        safetyNetLooper.start();
        super.onCreate();
    }
}
```

In manifest,
```xml
<application
    android:name=".MyApplication">
</application>
```

```java
SafetyNetHelper safetyNetHelper = new SafetyNetHelper(BuildConfig.API_KEY);
Handler handler = new Handler(MyApplication.INSTANCE.safetyNetLooper.getLooper());
GoogleApiClient googleApiClient = new GoogleApiClient.Builder(contex)
        .addApi(SafetyNet.API)
        .addConnectionCallbacks(googleApiConnectionCallbacks)
        .addOnConnectionFailedListener(googleApiConnectionFailedListener)
        .setHandler(handler) //Run on a new thread
        .build();
//Don't forget to connect!
googleApiClient.connect();
```

```Java
byte[] requestNonce = generateOneTimeRequestNonce();
SafetyNet.SafetyNetApi.attest(googleApiClient, requestNonce)
        .setResultCallback(new ResultCallback<SafetyNetApi.AttestationResult>() {

            @Override
            public void onResult(@NonNull SafetyNetApi.AttestationResult attestationResult) {
                Status status = attestationResult.getStatus();
                boolean isSuccess = status.isSuccess();
                if (!isSuccess)
                    callback.onFail(ErrorMessage.SAFETY_NET_API_NOT_WORK, ErrorMessage.SAFETY_NET_API_NOT_WORK.name());
                else {
                    try {
                        final String jwsResult = attestationResult.getJwsResult();
                        final JwsHelper jwsHelper = new JwsHelper(jwsResult);
                        final AttestationResult response = new AttestationResult(jwsHelper.getDecodedPayload());
                        if (!verifyJWSResponse) {
                            callback.onResponse(response.getFormattedString());
                            
                            //release SafetyNet HandlerThread
                            MyApplication.INSTANCE.safetyNetLooper.quit();
                        } else {
                            AndroidDeviceVerifier androidDeviceVerifier = new AndroidDeviceVerifier(ctx, jwsResult);
                            androidDeviceVerifier.verify(new AttestationTaskCallback() {
                                @Override
                                public void error(String errorMsg) {
                                    callback.onFail(ErrorMessage.FAILED_TO_CALL_GOOGLE_API_SERVICES, errorMsg);
                                    
                                    //release SafetyNet HandlerThread
                                    MyApplication.INSTANCE.safetyNetLooper.quit();
                                }

                                @Override
                                public void success(boolean isValidSignature) {
                                    if (isValidSignature)
                                        callback.onResponse("isValidSignature true\n\n" + response.getFormattedString());
                                    else
                                        callback.onFail(ErrorMessage.ERROR_VALID_SIGNATURE, ErrorMessage.ERROR_VALID_SIGNATURE.name());
                                
                                    //release SafetyNet HandlerThread
                                    MyApplication.INSTANCE.safetyNetLooper.quit();
                                }
                            });
                        }
                    } catch (JSONException e) {
                        callback.onFail(ErrorMessage.EXCEPTION, e.getMessage());
                        
                        //release SafetyNet HandlerThread
                        MyApplication.INSTANCE.safetyNetLooper.quit();
                    }
                }
            }
        });
```

- [SafetyNetUtils]
> Learn more about JWS : https://tools.ietf.org/html/rfc7515

## Step3. Call Attestation API to retrieve JWS messages
The JWS payloads I got by running this app on the real device and the nox monitor are a little different.

- On my mobile phone, ctsProfileMatch and basicIntegrity were both true.
```JSON
{
  "nonce":"pUkGirEXYOQefux33VWeSEmR0kBkLNGQaiQiZvE3VAc=",
  "timestampMs":1498814112718,
  "apkPackageName":"com.catherine.securitysample",
  "apkDigestSha256":"FPgrs1x05EaZiJkfKaitzEXTazg+GDDqYtbR5XyJiJE=",
  "ctsProfileMatch":true,
  "extension":"CbRP9k08+pZE",
  "apkCertificateDigestSha256":["9mLFS3eHWOBcHlA4MmODmfGvzgkbg2YSQ2z/ww9lCfw="],
  "basicIntegrity":true
}

```

- On a rooted one, ctsProfileMatch and basicIntegrity were both false.
```JSON
{
  "nonce":"FWypInssEmM+YBl61JCVPFx+bC5naGuIPQhkP3ait68=",
  "timestampMs":1502958413970,
  "apkPackageName":"",
  "apkDigestSha256":"",
  "ctsProfileMatch":false,
  "extension":"CdVwxgDa4bqk",
  "apkCertificateDigestSha256":"",
  "basicIntegrity":false
}
```

## Step4. Verify your JWS response (optional)
- First you must finish step1.
- **You can skip this step if you don't verify your attestation response from google APIs (I feel like this step is kind of like https validation. It probabily means man-in-the-middle attacks are allowed if you do not check the response.). Of course you can also validate the SSL certificate chain by yourself. Google highly recommends you to check your JWS statement.**
- **What "Android Device Verification API" dose is only checking JWS certificates and signatures. Its response (JSON payload) has nothing to do with the Android environments in which your app run.**
- I have this app call google Android Device Verification API until daily API queries exceed the quota limit. Then, instead of google server, the JWS response will be verified by devices. Here is a sample [AttestationAsyncTask].

>Follow these steps to verify the origin of the JWS message:
>1. Extract the SSL certificate chain from the JWS message.
>2. Validate the SSL certificate chain and use SSL hostname matching to verify that the leaf certificate was issued to the hostname attest.android.com.
>3. Use the certificate to verify the signature of the JWS message.


## Step5. Back to your application
- Post the JWS payload to your server to check the payload and return commands to your app.

If you want to read more about google security services for Android, you can watch [Google Security Services for Android : Mobile Protections at Google Scale], the youtube video. Or you could see my note [README_cn], they are almost the same.

Your workflow would be one of them:
1. (Security risk) Call Attestation APIs → Get a JWS response → Send JWS to your server → ?? - it depends on your server.
2. (Recommendation) Call Attestation APIs → Get a JWS response → Check the JWS response (step 4) → Send valid JWS to your server → ?? - it depends on your server.

# Warnings
As you add new secret keys, you must refill modulus, exponent and the other encrypted keys, because you'll get different RSA KeyPair (private key and public key) for every execution.

## Reference
- [Server Authentication During SSL Handshake]
- [Verifying a Certificate Chain]
- [JSON Web Signature (JWS) draft-jones-json-web-signature-01]

# License

```
Copyright 2017 Catherine Chen (https://github.com/Catherine22)

Licensed under the Apache License, Version 2.0 (the "License"); you may not
use this file except in compliance with the License. You may obtain a copy of
the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.
```

[RSAHelper]:<https://github.com/Catherine22/RSAHelper>
[MainActivity]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/MainActivity.java>
[Config.cpp]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/jni/Config.cpp>
[JNIHelper]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/JNIHelper.java>
[SafetyNet]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/safety_net>
[Android.mk]:<https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/jni/Android.mk>     
[Application.mk]:<https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/jni/Application.mk>
[NDK1]: https://github.com/Catherine22/MobileManager/blob/master/jni1.png  
[Google Security Services for Android : Mobile Protections at Google Scale]:<https://www.youtube.com/watch?v=exU1f_UBXGk>
[README_cn]:<https://github.com/Catherine22/SecuritySample/blob/master/README_cn.md>     
[AndroidDeviceVerifier]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/safety_net/AndroidDeviceVerifier.java>  
[AttestationAsyncTask]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/safety_net/AttestationAsyncTask.java>
[SafetyNetUtils]: <https://github.com/Catherine22/SecuritySample/blob/master/app/src/main/java/com/catherine/securitysample/safety_net/SafetyNetUtils.java>


[Server Authentication During SSL Handshake]:<https://docs.oracle.com/cd/E19693-01/819-0997/aakhc/index.html>
[Verifying a Certificate Chain]:<https://docs.oracle.com/cd/E19316-01/820-2765/gdzea/index.html>
[JSON Web Signature (JWS) draft-jones-json-web-signature-01]:<http://self-issued.info/docs/draft-jones-json-web-signature-01.html>
