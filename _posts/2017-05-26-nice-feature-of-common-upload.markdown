---
layout: post
title:  "Nice Stream API provided by Common Upload"
date: 2017-05-26 15:10:00 +0000
categories: Web
---

### Nice Stream API provided by Common Upload

My website needed support uploading file as large as 5 GB. After using the multi-part feature provided by Jesery, I managed to support it. But during real node test, we found it was a terrible solution because all the files would be buffered in dish before starting handle it.  It was not acceptable because the disk in our system is limited.  What made things worse is when the disk was full, no more action could release space except deleting those cache files and restarting tomcat. Hence I have to find another way to solve this issue.

### Apache Common File Upload

When I was trying to find a solution, the library common file upload came to me. After going thought the document,  I found it was perfect to solve my problem with streaming api which had been introduced in version 1.2.  I quote the words from Common File Upload  here to explain why people need such feature. 

<pre>
The traditional API, which is described in the User Guide, assumes, that file items must be stored 
somewhere, before they are actually accessible by the user. This approach is convenient, because it
allows easy access to an items contents. On the other hand, it is memory and time consuming.

The streaming API allows you to trade a little bit of convenience for optimal performance and and a
low memory profile. Additionally, the API is more lightweight, thus easier to understand
</pre>

```java
Usage
The usage of this API is straight forward. 

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
}
```


### What is form data  

Firstly, from the code I find it takes all the multi-parts as the stream and handle them one by one. Because A file upload request comprises an ordered list of items that are encoded according to [RFC 1867](http://www.ietf.org/rfc/rfc1867.txt). There is an example about how the form data looks like 

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

// Create the multipart stream to read input stream 
multi = new MultipartStream(input, boundary, notifier);
multi.setHeaderEncoding(charEncoding);

skipPreamble = true;

// put the pointer to the first item
findNextItem();


</pre>

From the code above, we can find it create a dedicated Stream class named "MultipartStream" to handle input stream. We have to understand how it works before understand how to boundary and read input stream. All the parameter needed can be found from the java doc, which is very clear so I do not need explain too much.


<pre>
   * @param input    The <code>InputStream</code> to serve as a data source.
     * @param boundary The token used for dividing the stream into
     *                 <code>encapsulations</code>.
     * @param bufSize  The size of the buffer to be used, in bytes.
     * @param pNotifier The notifier, which is used for calling the
     *                  progress listener, if any.
     *
     * @throws IllegalArgumentException If the buffer size is too small
     *
     * @since 1.3.1
     */
    public MultipartStream(InputStream input,
            byte[] boundary,
            int bufSize,
            ProgressNotifier pNotifier) {
</pre>

FileItemIterator heavily depend on this class to find findNextItem. I add some description in following attached code snippet to make it easier to understand. 

<pre>

/**
 * Called for finding the next item, if any.
 *
 * @return True, if an next item was found, otherwise false.
 * @throws IOException An I/O error occurred.
 */
private boolean findNextItem() throws IOException {
    ... Some preCheck Code...
    for (;;) { // loop until find result
        boolean nextPart;
        //Charlie: Skip to next next begining if found.
        if (skipPreamble) {   
            nextPart = multi.skipPreamble();
        } else {
            nextPart = multi.readBoundary();
        }
        //Charlie: If no more multipart, return false
        if (!nextPart) {
            if (currentFieldName == null) {
                // Outer multipart terminated -> No more data
                eof = true;
                return false;
            }
            // Inner multipart terminated -> Return to parsing the outer
            multi.setBoundary(boundary);
            currentFieldName = null;
            continue;
        }
       
        FileItemHeaders headers = getParsedHeaders(multi.readHeaders());
        if (currentFieldName == null) {
            // We're parsing the outer multipart
            //Charlie: seems job multipart can be nested in other multipart 
            String fieldName = getFieldName(headers);
            if (fieldName != null) {
                String subContentType = headers.getHeader(CONTENT_TYPE);
                if (subContentType != null
                        &&  subContentType.toLowerCase(Locale.ENGLISH)
                                .startsWith(MULTIPART_MIXED)) {
                    currentFieldName = fieldName;
                    // Multiple files associated with this field name
                    byte[] subBoundary = getBoundary(subContentType);
                    multi.setBoundary(subBoundary);
                    skipPreamble = true;
                    continue;
                }

                String fileName = getFileName(headers);
                //Charlie: Create a new dedicated stream for us to read content. 
                seems duplicated with following branch. 
                currentItem = new FileItemStreamImpl(fileName,
                        fieldName, headers.getHeader(CONTENT_TYPE),
                        fileName == null, getContentLength(headers));
                currentItem.setHeaders(headers);
                notifier.noteItem();
                itemValid = true;
                return true;
            }
        } else {
            String fileName = getFileName(headers);
            if (fileName != null) {
                currentItem = new FileItemStreamImpl(fileName,
                        currentFieldName,
                        headers.getHeader(CONTENT_TYPE),
                        false, getContentLength(headers));
                currentItem.setHeaders(headers);
                notifier.noteItem();
                itemValid = true;
                return true;
            }
        }
        multi.discardBodyData();
    }
}

</pre>

So far all the main process of steam API has been introduced, hope it could help us to understand how it works. Forgive my broken English anyway. 


### Reference 

- [Apache Commons File Upload: ](https://commons.apache.org/proper/commons-fileupload/streaming.html)


