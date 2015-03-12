# Amazon S3 Uploader
__Fork of leozepo:S3 allowing to upload files in array.__  

__This package is no longer supporting meteor releases before 0.9.0.__ S3 provides a simple way for uploading files to the Amazon S3 service with a progress bar. This is useful for uploading images and files that you want accesible to the public. S3 is built on [Knox](https://github.com/LearnBoost/knox), a module that becomes available server-side after installing this package.

## Installation

``` sh
$ meteor add khamoud:s3
```

## How to use

### Step 1
Define your Amazon S3 credentials. SERVER SIDE.

``` javascript
S3.config = {
	key: 'amazonKey',
	secret: 'amazonSecret',
	bucket: 'bucketName'
};
```

### Step 2
Create a file input and progress indicator. CLIENT SIDE.

``` handlebars
<template name="s3_tester">
	<input type="file" class="file_bag">
	<button class="upload">Upload</button>

	{{#each files}}
		<p>{{percent_uploaded}}</p>
	{{/each}}
</template>
```

### Step 3
Create a function to upload the files and a helper to see the uploads progress. CLIENT SIDE.

``` javascript
Template.s3_tester.events({
	"click button.upload": function(){
		var files = $("input.file_bag")[0].files
		S3.upload(files,"/subfolder",function(e,r){
			console.log(r);
		});
	}
})

Template.s3_tester.helpers({
	"files": function(){
		return S3.collection.find();
	}
})
```

## Using Canvas Elements
### Step 1
Add a file input and canvas element CLIENT SIDE
``` html
<div class="container">
    <canvas id="canvas"></canvas>
</div>
<div>
	<input class="fileBag" type="file">
</div>
```

### Step 2
Upon selecting an image you want to draw the image onto a canvas element with Javascript CLIENT SIDE
``` javascript

"change .fileBag": function(event){
    var preview = event.target;
    readURL(preview);
}
```
...
``` javascript
function readURL(input) {
    if (input.files && input.files[0]) {
        var reader = new FileReader();
        reader.onload = function (e) {
            var img = new Image();
            var canvas = document.getElementById("canvas");
            var ctx = canvas.getContext("2d");
            img.onload = function() {
                ctx.drawImage(canvas, 0, 0, canvas.width, canvas.height, 0, 0, canvas.width, canvas.height);
            }
            img.src = e.target.result;
        }
        reader.readAsDataURL(input.files[0]);
    }
}
```
### Step 3 (Final step)
Implement a function to convert a dataURI to a blob CLIENT SIDE (e.g)
``` javascript
function dataURItoBlob(dataURI) {
    var binary = atob(dataURI.split(',')[1]);
    var array = [];
    for(var i = 0; i < binary.length; i++) {
        array.push(binary.charCodeAt(i));
    }
    var uid = Date.now() + ".jpg";
    return new File([new Uint8Array(array)],uid, {type: 'image/jpeg'});
}
```

Manipulate the image however you want then get the data uri back when ready to upload. CLIENT SIDE  
__IMPORTANT__: The new blob element must be put into an array before uploading
``` javascript
'submit #uploadPictureForm':function(e){
	e.preventDefault();
	var url = document.getElementById("canvas").toDataURL();
	var blob = dataURItoBlob(url);
	var picArr = [blob]
	S3.upload(picArr, "/blobFolder", function(err,res){
		// Do stuff
	})
}
```


## Create your Amazon S3
For all of this to work you need to create an aws account. On their website create navigate to S3 and create a bucket. Navigate to your bucket and on the top right side you'll see your account name. Click it and go to Security Credentials. Once you're in Security Credentials create a new access key under the Access Keys (Access Key ID and Secret Access Key) tab. This is the info you will use for the first step of this plug. Go back to your bucket and select the properties OF THE BUCKET, not a file. Under Static Website Hosting you can Enable website hosting, to do that first upload a blank index.html file and then enable it. YOU'RE NOT DONE.

You need to set permissions so that everyone can see what's in there. Under the Permissions tab click Edit CORS Configuration and paste this:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
    </CORSRule>
</CORSConfiguration>
```

Save it. Now click Edit bucket policy and paste this, REPLACE THE BUCKET NAME WITH YOUR OWN:

``` javascript
{
	"Version": "2008-10-17",
	"Statement": [
		{
			"Sid": "AllowPublicRead",
			"Effect": "Allow",
			"Principal": {
				"AWS": "*"
			},
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::YOURBUCKETNAMEHERE/*"
		}
	]
}
```

Enjoy, this took me a long time to figure out and I'm sharing it so that nobody has to go through all that.
__NOTE:__ It might take a couple of hours before you can actually start uploading to S3. Amazon takes some time to make things work.

## API

### S3 (CLIENT SIDE)

#### S3.collection
This is a null Meteor.Collection that exists only on the users client. After the user leaves the page or refreshes, the collection disappears forever.

#### S3.stream
This is the meteor stream that is created between the server and the S3.collection object on the client to relay information. You probably don't need access to this.

#### S3.upload(files,path,callback)
This is the upload function that manages all the dramatic things you need to do for something so essentially simple.

__Parameters:__
*	__files:__ Must be a FileList object or an array of files. You can get this via jQuery via $("input[type='file']")[0].files
*	__path:__ Must be in this format ("/folder/other_folder"). So basically always start with "/" and never end with "/". This is required.
*	__callback:__ A function that is run after the upload is complete returning an Error as the first parameter (if there is one), and a Result as the second.
*	__Result:__ The returned value of the callback function if there is no error. It returns an object with these keys:
	*	__total_uploaded:__ Integer (bytes)
	*	__percent_uploaded:__ Integer (out of 100)
	*	__uploading:__ Boolean (false if done uploading)
	*	__url:__ String (S3 hosted URL)
	*	__secure_url:__ String (S3 hosted URL for https)

#### S3.delete(path,callback)
This function permanently destroys files located in your S3 bucket. It still needs more work for security in the form of allow/deny rules.

__Parameters:__
*	__path:__ Must be in this format ("/folder/other_folder/file.extension"). So basically always start with "/" and never end with "/". This is required.
*	__callback:__ A function that is run after the upload is complete returning an Error as the first parameter (if there is one), and a Result as the second.

### S3 (SERVER SIDE)

#### S3.config
This is where you define your key, secret, and bucket.

``` javascript
S3.config = {
	key: 'amazonKey',
	secret: 'amazonSecret',
	bucket: 'bucketName'
};
```

#### S3.stream
This is the meteor stream that is created between the server and the S3.collection object on the client to relay information. You probably don't need access to this.

#### S3.knox
The current knox client.


#### Developer Notes
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/frames.html
https://github.com/Differential/meteor-uploader/blob/master/lib/UploaderFile.coffee#L169-L178
