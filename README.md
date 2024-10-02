# Android-Photo-Upload-S3

Create the code for an Android project that allows for uploading photos taken by the Android phone, to an AWS S3 Bucket.
Certainly! Here's a basic Android project structure and code that allows for uploading photos taken by the Android phone to an AWS S3 bucket. This example uses the AWS SDK for Android.

First, you'll need to set up your Android project and add the necessary dependencies:

1. In your app's `build.gradle` file, add the following dependencies:

```gradle
dependencies {
    implementation 'com.amazonaws:aws-android-sdk-s3:2.19.0'
    implementation 'com.amazonaws:aws-android-sdk-mobile-client:2.19.0'
}
```

2. Create a new Android project and add the following files:

`MainActivity.java`:

```java
package com.example.s3photoupload;

import android.Manifest;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Bundle;
import android.provider.MediaStore;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;
import androidx.core.content.FileProvider;

import java.io.File;
import java.io.IOException;

public class MainActivity extends AppCompatActivity {

    private static final int REQUEST_IMAGE_CAPTURE = 1;
    private static final int REQUEST_PERMISSION_CAMERA = 2;
    private Uri photoUri;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button takePictureButton = findViewById(R.id.take_picture_button);
        takePictureButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.permission.CAMERA)
                        != PackageManager.PERMISSION_GRANTED) {
                    ActivityCompat.requestPermissions(MainActivity.this,
                            new String[]{Manifest.permission.CAMERA},
                            REQUEST_PERMISSION_CAMERA);
                } else {
                    dispatchTakePictureIntent();
                }
            }
        });
    }

    private void dispatchTakePictureIntent() {
        Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
            File photoFile = null;
            try {
                photoFile = createImageFile();
            } catch (IOException ex) {
                Toast.makeText(this, "Error creating image file", Toast.LENGTH_SHORT).show();
            }

            if (photoFile != null) {
                photoUri = FileProvider.getUriForFile(this,
                        "com.example.s3photoupload.fileprovider",
                        photoFile);
                takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoUri);
                startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);
            }
        }
    }

    private File createImageFile() throws IOException {
        String timeStamp = String.valueOf(System.currentTimeMillis());
        String imageFileName = "JPEG_" + timeStamp + "_";
        File storageDir = getExternalFilesDir(null);
        return File.createTempFile(imageFileName, ".jpg", storageDir);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
            uploadPhotoToS3();
        }
    }

    private void uploadPhotoToS3() {
        S3Uploader.uploadFile(this, photoUri);
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == REQUEST_PERMISSION_CAMERA) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                dispatchTakePictureIntent();
            } else {
                Toast.makeText(this, "Camera permission is required", Toast.LENGTH_SHORT).show();
            }
        }
    }
}
```

`S3Uploader.java`:

```java
package com.example.s3photoupload;

import android.content.Context;
import android.net.Uri;
import android.util.Log;

import com.amazonaws.auth.CognitoCachingCredentialsProvider;
import com.amazonaws.mobileconnectors.s3.transferutility.TransferListener;
import com.amazonaws.mobileconnectors.s3.transferutility.TransferObserver;
import com.amazonaws.mobileconnectors.s3.transferutility.TransferState;
import com.amazonaws.mobileconnectors.s3.transferutility.TransferUtility;
import com.amazonaws.regions.Region;
import com.amazonaws.regions.Regions;
import com.amazonaws.services.s3.AmazonS3Client;

import java.io.File;

public class S3Uploader {

    private static final String TAG = "S3Uploader";
    private static final String COGNITO_POOL_ID = "YOUR_COGNITO_POOL_ID";
    private static final String BUCKET_NAME = "YOUR_BUCKET_NAME";
    private static final String COGNITO_POOL_REGION = "YOUR_COGNITO_POOL_REGION";

    public static void uploadFile(final Context context, Uri fileUri) {
        CognitoCachingCredentialsProvider credentialsProvider = new CognitoCachingCredentialsProvider(
                context,
                COGNITO_POOL_ID,
                Regions.fromName(COGNITO_POOL_REGION)
        );

        AmazonS3Client s3Client = new AmazonS3Client(credentialsProvider);
        s3Client.setRegion(Region.getRegion(Regions.fromName(COGNITO_POOL_REGION)));

        TransferUtility transferUtility = TransferUtility.builder()
                .context(context)
                .awsConfiguration(AWSMobileClient.getInstance().getConfiguration())
                .s3Client(s3Client)
                .build();

        File file = new File(fileUri.getPath());
        String key = file.getName();

        TransferObserver uploadObserver = transferUtility.upload(
                BUCKET_NAME,
                key,
                file
        );

        uploadObserver.setTransferListener(new TransferListener() {
            @Override
            public void onStateChanged(int id, TransferState state) {
                if (TransferState.COMPLETED == state) {
                    Log.d(TAG, "Upload completed!");
                }
            }

            @Override
            public void onProgressChanged(int id, long bytesCurrent, long bytesTotal) {
                float percentDonef = ((float) bytesCurrent / (float) bytesTotal) * 100;
                int percentDone = (int) percentDonef;
                Log.d(TAG, "ID:" + id + " bytesCurrent: " + bytesCurrent + " bytesTotal: " + bytesTotal + " " + percentDone + "%");
            }

            @Override
            public void onError(int id, Exception ex) {
                Log.e(TAG, "Error during upload: " + ex.getMessage());
            }
        });
    }
}
```

`activity_main.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/take_picture_button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Take Picture"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

3. Update your `AndroidManifest.xml` to include necessary permissions and the FileProvider:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.s3photoupload">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">

        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="com.example.s3photoupload.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>

    </application>

</manifest>
```

4. Create a new XML resource file named `file_paths.xml` in the `res/xml` directory:

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-path name="my_images" path="Android/data/com.example.s3photoupload/files/Pictures" />
</paths>
```

Remember to replace `YOUR_COGNITO_POOL_ID`, `YOUR_BUCKET_NAME`, and `YOUR_COGNITO_POOL_REGION` in the `S3Uploader.java` file with your actual AWS credentials and settings.

This code provides a basic structure for an Android app that can take pictures and upload them to an S3 bucket. The app requests camera permissions, takes a picture, and then uploads it to the specified S3 bucket using the AWS SDK for Android.

Note that this is a basic implementation and you may want to add error handling, progress indicators, and other features to make it more robust and user-friendly.
