---
layout: post
title:  "Nice Stream API provided by Common Upload"
date: 2017-05-126 15:10:00 +0000
categories: Web
---

### Nice Stream API provided by Common Upload

My website needed support uploading  file as large as 5 GB. After using the mulit-part feature provided by Jesery, I managed to support this feature. But during real node test, we found it was a terrible solution because all the file would be cached in dish before finishing handle it.  It was not acceptable because the disk in our system is limited.  What made things worse is when the disk was full, no more action could release space except deleting those cache files and restarting tomcat. Hence I have to find another way to solve this issue.

### Apache Common File Upload

When I was trying to find a solution, the library common file upload came to me. After go thought the document,  I found it was perfect to solve my problem with streaming api which had been introduced in version 1.2.  I quote the words from Common File Upload  here to explain why people need such feature. 

<pre>
The traditional API, which is described in the User Guide, assumes, that file items must be stored 
somewhere, before they are actually accessible by the user. This approach is convenient, because it
allows easy access to an items contents. On the other hand, it is memory and time consuming.

The streaming API allows you to trade a little bit of convenience for optimal performance and and a
low memory profile. Additionally, the API is more lightweight, thus easier to understand
</pre>


### Usage
The useage of  this API is straight forward. 

<pre>
try {
    iter = upload.getItemIterator(req);
    while (iter.hasNext()) {
        FileItemStream item = iter.next();
        String name = item.getFieldName();
        System.out.println("open stream ");

        int i = 0;
        InputStream stream = item.openStream();
        if (item.isFormField()) {
            System.out.println("File field " + name + " with file name "
                    + item.getName() + " detected.");

        } else {
            System.out.println("File field " + name + " with file name "
                    + item.getName() + " detected.");
            try (BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(stream))) {
                for (String line = bufferedReader.readLine(); line != null; line = bufferedReader.readLine()) {
                    if(i++ % 10000 ==0){
                        System.out.print(line);
                    }
                }
            }
        }

        System.out.println("finish");
    }
} catch (FileUploadException e) {
    e.printStackTrace();
}</pre>


### What is form data  

Firstly, from the code I find it takes all the multi parts as the stream and handle them one by one. Because A file upload request comprises an ordered list of items that are encoded according to [RFC 1867](http://www.ietf.org/rfc/rfc1867.txt). There is an example about how the form data looks like 

Suppose the server supplies the following page:

 <FORM ACTION="http://server.dom/cgi/handle"
       ENCTYPE="multipart/form-data"
       METHOD=POST>
 What is your name? <INPUT TYPE=TEXT NAME=submitter>
 What files are you sending? <INPUT TYPE=FILE NAME=pics>
 </FORM>

and the user types "Joe Blow" in the name field, and selects a text
file "file1.txt" for the answer to 'What files are you sending?'

The client might send back the following data:

<pre>

Content-type: multipart/form-data, boundary=AaB03x

--AaB03x
content-disposition: form-data; name="field1"

Joe Blow
--AaB03x
content-disposition: form-data; name="pics"; filename="file1.txt"
Content-Type: text/plain

 ... contents of file1.txt ...
--AaB03x--

</pre>


### What Apache Common File Upload has done 
I am trying to add my understanding of the source code to explain what this library has done and make our life much easier. In order to handle form data, library need to handle HTTP stream directly which making things much trickier and fallible comparing to using standard implementation like Jersey. 

The First task of File upload is to find the boundary of each filed and populate the file iterator. 
I can't paste all the code here but pick up some code snippet and try to explain it based on my understanding. The pseudocode looks like 

<pre>
..Precheck code ...
InputStream input = ctx.getInputStream();
final int contentLengthInt = ctx.getContentLength();
final long requestSize = UploadContext.class.isAssignableFrom(ctx.getClass())
                         // Inline conditional is OK here CHECKSTYLE:OFF
                         ? ((UploadContext) ctx).contentLength()
                         : contentLengthInt;
                         // CHECKSTYLE:ON
..Check request size...

boundary = getBoundary(contentType);
// In order to notifier client the progress of upload 
notifier = new MultipartStream.ProgressNotifier(listener, requestSize);

// Create the multipart stream
multi = new MultipartStream(input, boundary, notifier);
multi.setHeaderEncoding(charEncoding);

skipPreamble = true;

// put the pointer to the first item
findNextItem();


</pre>
