# Upload file to OSS (JAVA)

At this section we actually need 3 features:

1. Create buckets
2. List buckets & objects (files)
3. Upload objects (files)

## OSS.java

Create a new Java Class with the following content. This file handles creating bucket and listing buckets.

```java
import java.io.*;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;
import javax.xml.bind.DatatypeConverter;

import org.json.*;

import com.autodesk.client.ApiException;
import com.autodesk.client.ApiResponse;
import com.autodesk.client.api.*;
import com.autodesk.client.model.*;

@WebServlet({"/oss"})
public class oss extends HttpServlet {

    public oss() {
    }

    public void init() throws ServletException {

    }

    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        String id = req.getParameter("id");
        resp.setCharacterEncoding("utf8");
        resp.setContentType("application/json");
        try
        {
            String internalToken = oauth.getTokenInternal();


            if (id.equals("#")) {// root
                BucketsApi bucketsApi = new BucketsApi();

                //no idea how to set startAt. looks 'abc' can workaround
                ApiResponse<Buckets> buckets = bucketsApi.getBuckets("us",100,"abc",oauth.OAuthClient(null),oauth.getCredentials());

                JSONArray bucketsArray = new JSONArray();
                PrintWriter out = resp.getWriter();

                for(int i=0;i<buckets.getData().getItems().size();i++){


                    BucketsItems eachItem = buckets.getData().getItems().get(i);
                    JSONObject obj = new JSONObject();

                    obj.put("id", eachItem.getBucketKey());
                    obj.put("text", eachItem.getBucketKey());
                    obj.put("type", "bucket");
                    obj.put("children", true);

                    bucketsArray.put(obj);

                }

                out.print(bucketsArray);

            }
            else
            {

                // as we have the id (bucketKey), let's return all objects
                ObjectsApi objectsApi = new ObjectsApi();

                ApiResponse<BucketObjects> objects = objectsApi.getObjects(id,100,
                        null,null,
                        oauth.OAuthClient(null),oauth.getCredentials());


                JSONArray objectsArray = new JSONArray();
                PrintWriter out = resp.getWriter();

                for(int i=0;i<objects.getData().getItems().size();i++){


                    ObjectDetails eachItem = objects.getData().getItems().get(i);
                    String base64Urn = DatatypeConverter.printBase64Binary(eachItem.getObjectId().getBytes());


                    JSONObject obj = new JSONObject();

                    obj.put("id", base64Urn);
                    obj.put("text", eachItem.getObjectKey());
                    obj.put("type", "object");
                    obj.put("children", false);


                    objectsArray.put(obj);

                }

                out.print(objectsArray);


            }
        }catch (ApiException autodeskExp) {
            System.out.print("get buckets & objects exception: "+ autodeskExp.toString());
            resp.setStatus(500);

        }
         catch(Exception exp){
             System.out.print("get buckets & objects exception: "+ exp.toString());
             resp.setStatus(500);
         }

    }

    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        try {

            //from https://stackoverflow.com/questions/3831680/httpservletrequest-get-json-post-data/3831791

            StringBuffer jb = new StringBuffer();
            String line = null;
            try {
                BufferedReader reader = req.getReader();
                while ((line = reader.readLine()) != null)
                    jb.append(line);
            } catch (Exception e) { /*report an error*/ }

            // Create a new bucket
            try
            {
                 JSONObject jsonObject = new JSONObject(jb.toString());

                String bucketKey = jsonObject.getString("bucketKey");
                String internalToken = oauth.getTokenInternal();
                BucketsApi bucketsApi = new BucketsApi();
                PostBucketsPayload postBuckets = new PostBucketsPayload();
                postBuckets.setBucketKey(bucketKey);
                postBuckets.setPolicyKey(PostBucketsPayload.PolicyKeyEnum.TRANSIENT);// expires in 24h

                ApiResponse<Bucket> newbucket = bucketsApi.createBucket(postBuckets, null,
                        oauth.OAuthClient(null),oauth.getCredentials());

                resp.setStatus(200);


            }
            catch (ApiException autodeskExp) {
                System.out.print("get buckets & objects exception: "+ autodeskExp.toString());
                resp.setStatus(500);

            }
            catch(Exception exp){
                System.out.print("get buckets & objects exception: "+ exp.toString());
                resp.setStatus(500);

            }

        } catch (JSONException e) {
            // crash and burn
            throw new IOException("Error parsing JSON request string");
        }

    }

    public void destroy() {
        super.destroy();
    } 
}
```



As we plan to suppor the [jsTree](https://www.jstree.com/) library, our **GET oss/buckets** need to return handle the `id` querystring parameter and return buckets when `id=#` and objects for a given bucketKey passed as `id=bucketKey`.

## OSSUploads.java

Create a new Java Class with the following content. This file handles uploading file. The workflow gets the file stream and uploads to Forge.

```java
import java.io.*;
import java.util.Iterator;
import java.util.List;
import java.util.regex.*;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.fileupload.FileItem;
import org.apache.commons.fileupload.disk.DiskFileItemFactory;
import org.apache.commons.fileupload.servlet.ServletFileUpload;

import com.autodesk.client.ApiException;
import com.autodesk.client.ApiResponse;
import com.autodesk.client.api.ObjectsApi;
import com.autodesk.client.model.ObjectDetails;

@WebServlet({"/ossuploads"})
public class ossuploads  extends HttpServlet {

    public ossuploads() {
    }

    public void init() throws ServletException {

    }

    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

    }

    private String filename(String contentTxt) throws UnsupportedEncodingException {
        Pattern pattern = Pattern.compile("filename=\"(.*)\"");
        Matcher matcher = pattern.matcher(contentTxt);
        matcher.find();
        return matcher.group(1);
    }

    private byte[] bodyContent(HttpServletRequest request) throws IOException {
        try(ByteArrayOutputStream out = new ByteArrayOutputStream()) {
            InputStream in = request.getInputStream();
            byte[] buffer = new byte[1024];
            int length = -1;
            while((length = in.read(buffer)) != -1) {
                out.write(buffer, 0, length);
            }
            return out.toByteArray();
        }
    }
    protected void doPost(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException,FileNotFoundException  {

        //from https://stackoverflow.com/questions/3831680/httpservletrequest-get-json-post-data/3831791
        try
        { 
            //from https://stackoverflow.com/questions/13048939/file-upload-with-servletfileuploads-parserequest
            if (!ServletFileUpload.isMultipartContent(req)) { 
                //not multiparts/formdata
                res.setStatus(500);
            }
            else
            {
                 String bucketKey = "";
                String filename="";
                String filepath = "/fileuploads";

                List<FileItem> items = new ServletFileUpload(new DiskFileItemFactory()).parseRequest(req);
                Iterator iter = items.iterator();

                File fileToUpload = null;

                while (iter.hasNext()) {
                    FileItem item = (FileItem) iter.next();

                    if (!item.isFormField()) {
                        filename = item.getName();

                        String root = getServletContext().getRealPath("/");
                        File path = new File(root + filepath);
                        if (!path.exists()) {
                            boolean status = path.mkdirs();
                        }

                        filepath = path + "/" + filename;
                        fileToUpload = new File(filepath);
                        item.write(fileToUpload);
                     }
                    else
                    {
                        if(item.getFieldName().equals("bucketKey")){
                            bucketKey = item.getString();
                        }
                     }
                }

                ObjectsApi objectsApi = new ObjectsApi();

                //this call will cause the issue below.
                //working on the problem
                //the codes are for stackholder.
                //com.sun.jersey.api.client.ClientHandlerException: com.sun.jersey.api.client.ClientHandlerException: A message body writer for Java type, class [B, and MIME media type, application/octet-stream, was not found
                ApiResponse<ObjectDetails> response =
                        objectsApi.uploadObject(bucketKey, filename,
                                (int)fileToUpload.length(),
                                fileToUpload, null, null,
                                oauth.OAuthClient(null),oauth.getCredentials());


                res.setStatus(response.getStatusCode());
            }

        }
        catch(ApiException adskexp){

        }catch(FileNotFoundException fileexp){
            System.out.print("get buckets & objects exception: "+ fileexp.toString());

        }

        catch(Exception exp){
            System.out.print("get buckets & objects exception: "+ exp.toString());

        }
    }
    public void destroy() {
        super.destroy();
    }
}
```

Explictly expose the endpoint in `/web/WEB-INF/web.xml`:
```xml

    <servlet>
            <servlet-name>oss</servlet-name>
            <servlet-class>oss</servlet-class>
    </servlet>

    <servlet-mapping>
            <servlet-name>oss</servlet-name>
            <url-pattern>/api/forge/oss/buckets</url-pattern>
    </servlet-mapping>

    <servlet>
            <servlet-name>ossuploads</servlet-name>
            <servlet-class>ossuploads</servlet-class>
    </servlet>

    <servlet-mapping>
            <servlet-name>ossuploads</servlet-name>
            <url-pattern>/api/forge/oss/objects</url-pattern>
    </servlet-mapping>
```

 

Note how we reuse the `/src/resources/oauth.java` file to call `.getTokenInternal()` on all functions. 

!> Upload a file from the client (browser) directly to Autodesk Forge is possible, but requires giving the client a **write-enabled** access token, which is **NOT SECURE**.

Next: [Translate the file](modelderivative/translate/)