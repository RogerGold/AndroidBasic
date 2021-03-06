# Sharing Content between Android apps

Sharing is caring, as they say, but sharing on Android means something perhaps slightly different. 
‘Sharing’ is really shorthand for sending content such as text, formatted text, files, or images between apps.

### Sharing text
Sharing plain text is, as you might imagine, a good place to start. 
In fact, there’s not a whole lot to it:

    Intent shareIntent = ShareCompat.IntentBuilder.from(activity)
      .setType("text/plain")
      .setText(shareText)
      .getIntent();
    if (shareIntent.resolveActivity(getPackageManager()) != null) {
      startActivity(shareIntent);
    }
    
### Sharing HTML text
Some apps, most notably email clients, also support formatting with HTML. 
The changes, compared to plain text, are fairly minor:

    Intent shareIntent = ShareCompat.IntentBuilder.from(activity)
      .setType("text/html")
      .setHtmlText(shareHtmlText)
      .setSubject("Definitely read this")
      .addEmailTo(importantPersonEmailAddress)
      .getIntent();
      
### Receiving text
While the focus so far has been on the sending side, 
it is helpful to know exactly what is happening on the other side 
(if not just to build a simple receiving app to install on your emulator for testing purposes). 
Receiving Activities add an intent filter to the Activity:

    <activity android:name=”.ShareActivity”>
      <intent-filter>
        <action android:name=”android.intent.action.SEND”/>
        <category android:name=”android.intent.category.DEFAULT”/>
        <category android:name=”android.intent.category.BROWSABLE”/>
        <data android:mimeType=”text/plain”/>
      </intent-filter>
    </activity>
    
Note: In order to receive implicit intents, you must include the CATEGORY_DEFAULT category in the intent filter. 
The methods startActivity() and startActivityForResult() treat all intents as if they declared the CATEGORY_DEFAULT category. 
If you do not declare it in your intent filter, no implicit intents will resolve to your activity.

And to actually extract the information from the Intent, the useful [ShareCompat.IntentReader](http://developer.android.com/reference/android/support/v4/app/ShareCompat.IntentReader.html?utm_campaign=android_series_sharingcontent_012116&utm_source=medium&utm_medium=blog) can be used:

    ShareCompat.IntentReader intentReader =
        ShareCompat.IntentReader.from(activity);
    if (intentReader.isShareIntent()) {
      String[] emailTo = intentReader.getEmailTo();
      String subject = intentReader.getSubject();
      String text = intentReader.getHtmlText();
      // Compose an email
    }
  
### Sharing files and images
While sending and receiving text is straightforward enough (create text, include it in Intent), 
sending files (and particularly images — the most common type by far) has an additional wrinkle:
file permissions.The simplest code you might try might look like

    File imageFile = ...;
    Uri uriToImage = ...; // Convert the File to a Uri
    Intent shareIntent = ShareCompat.IntentBuilder.from(activity)
      .setType("image/png")
      .setStream(uriToImage)
      .getIntent();
      
And that almost works — the tricky part is in getting a Uri to the File that other apps can actually read, 
particularly when it comes to Android 6.0 Marshmallow devices and runtime permissions 
(which include the now dangerous READ_EXTERNAL_STORAGE and WRITE_EXTERNAL_STORAGE permissions).

My plea: don’t use Uri.fromFile(). It forces receiving apps to have the READ_EXTERNAL_STORAGE permission, 
won’t work at all if you are trying to share across users, and prior to KitKat, would require your app to have
WRITE_EXTERNAL_STORAGE. And really important share targets, like Gmail, won’t have the 
READ_EXTERNAL_STORAGE permission — so it’ll just fail.

Instead, you can use URI permissions to grant other apps access to specific Uris.
While URI permissions don’t work on file:// URIs as is generated by Uri.fromFile(), 
they do work on Uris associated with Content Providers. Rather than implement your own just for this, 
you can and should use FileProvider as explained in the File Sharing Training.

Once you have it set up, our code becomes:

    File imageFile = ...;
    Uri uriToImage = FileProvider.getUriForFile(
        context, FILES_AUTHORITY, imageFile);
    Intent shareIntent = ShareCompat.IntentBuilder.from(activity)
      .setStream(uriToImage)
      .getIntent();
    // Provide read access
    shareIntent.setData(uriToImage);
    shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    
 Using [FileProvider.getUriForFile()](http://developer.android.com/reference/android/support/v4/content/FileProvider.html?utm_campaign=android_series_sharingcontent_012116&utm_source=medium&utm_medium=blog#getUriForFile%28android.content.Context,%20java.lang.String,%20java.io.File%29), you’ll get a Uri actually suitable for sending to another app 
 — they’ll be able to read it without any storage permissions — instead, you are specifically granting 
 them read permission with FLAG_GRANT_READ_URI_PERMISSION.
 
 Note: we don’t call setType() anywhere when building our ShareCompat 
 (even though in the video I did set it). As explained in the [setDataAndType() Javadoc](http://developer.android.com/reference/android/content/Intent.html?utm_campaign=android_series_sharingcontent_012116&utm_source=medium&utm_medium=blog#setDataAndType%28android.net.Uri,%20java.lang.String%29), 
 the type is automatically inferred from the data URI using getContentResolver().getType(uriToImage). 
 Since FileProvider returns the correct mime type automatically, we don’t need to manually specify a mime type at all.
 
### Receiving files
Receiving files isn’t too different from text because you’re still going to use ShareCompat.IntentReader.
For example, to make a Bitmap out of an incoming file, it would look like:

    Uri uri = ShareCompat.IntentReader.from(activity).getStream();
    Bitmap bitmap = null;
    try {
      // Works with content://, file://, or android.resource:// URIs
      InputStream inputStream =
          getContentResolver().openInputStream(uri);
      bitmap = BitmapFactory.decodeStream(inputStream);
    } catch (FileNotFoundException e) {
      // Inform the user that things have gone horribly wrong
    }
 
 Of course, you’re free to do whatever you want with the InputStream — watch out for images that are so large you hit an OutOfMemoryException. 
 All of the things you know about [loading Bitmaps](http://developer.android.com/training/displaying-bitmaps/load-bitmap.html?utm_campaign=android_series_sharingcontent_012116&utm_source=medium&utm_medium=blog) still apply.
 
 ref: [ShareCompat](https://developer.android.com/reference/android/support/v4/app/ShareCompat.html?utm_campaign=android_series_sharingcontent_012116&utm_source=medium&utm_medium=blog)
 
 ref:[FileProvider](https://developer.android.com/reference/android/support/v4/content/FileProvider.html?utm_campaign=android_series_sharingcontent_012116&utm_source=medium&utm_medium=blog)

ref :[youtube](https://www.youtube.com/watch?v=C28pvd2plBA)
