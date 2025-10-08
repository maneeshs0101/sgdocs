Upload a new content resulting in a new document version. To be used from SG network only
Creates a new version of the document with the new uploaded mimetype content. If the mimetype already exists, it is replaced by the new one in the new version, except if the documents belongs to a WORM model. The old mimetype remains available in the old version.
About rightId, confidentialityLevel and destroyDate:

If not specified values are copied from previous versions.
If specified values are set on the new version
To apply default configuration from the model, send empty values
Uploads document content for a given mimeType, it creates a new version of the document.
There are two ways to upload a file :

In a single request. This approach is usually better, since it requires fewer requests and thus has better performance. If the content length is greater than 50MB, we recommend you to use multiple chunks. If the content length is greater than 500MB, the request will be rejected with 413 status code.
Use multiple chunk approach if:
Your request origin is from lan
You transfer large files (> 50MB)
The likelihood of a network interruption or some other transmission failure is high (for example, if you upload a file from a mobile app).
You need to reduce the amount of data transferred in any single request. You might need to do this when there is a fixed time limit for individual requests or to reduce consumed bandwith.
You need to provide a customized indicator to show the upload progress.
The behavior of this resource is highly inspired by Multiple chunks upload on Google drive APIs

Authorizations:
sgConnect
path Parameters
workspaceId
required
string
Example: c85df01e-4f02-4f23-802e-57bd29ec3f4f
Workspace Id

documentId
required
string
Example: 1d9d69d8-ff0d-4774-8689-af39c3be4104
Document Id

query Parameters
rightId	
string
Example: rightId=d2f89d53-c79e-4bc8-9f03-592ca9c3ff42
Optional right id. It must be a Right of type DOCUMENT_VERSION

confidentialityLevel	
string
Example: confidentialityLevel=C1
The confidentiality level of the document according to the Societe General Policy. If provided overrides the default confidentiality level set on the model

destroyDate	
string <date>
Example: destroyDate=2025-01-01
The date when the document must be destroyed. If provided overrides the destroy date calculated from the model default expiry

contentsOutsideSGNetwork	
boolean
Indicate if content is available outside SG Network or not.

header Parameters
x-audit-comment	
string
Example: my comment
Allow to send comment in audit event

Content-Type
required
string
Example: application/pdf
Content mimetype to be uploaded

Content-Length
required
number
Example: 123456
Content size. Set to the number of bytes in the current chunk or in the whole file if the upload is not chunked

Content-Disposition
required
string
Example: Content-Disposition: binary;filename=example.pdf
Set to the the file name. For example, Content-Disposition: binary;filename="example.pdf"

X-Content-Sha256	
string
Example: 5f612104c7d7276e6c5b22c5635ef07fddaa96f80ec605c19f005cfa21edfa9e
Provide a hexadecimal string representing sha256 content hash if you want to ensure content integrity

Content-Range	
string
Example: Content-Range: bytes 0-524287/2000000
Multiple Chunks Upload : Set to show which bytes in the file you upload. For example, Content-Range: bytes 0-524287/2000000 shows that you upload the first 524,288 bytes (256 x 1024 x 2) in a 2,000,000 byte file.

In case of failure or if you don't know from where upload should be continued you can set this header to bytes *, you will then get the current status of your chunked uploads (201 status code if the upload is completed or 200 if you need to transfer more bytes)

You can also send reset to reset a chunked upload to it's initial stage if you wan't to cancel your upload

Request Body schema: */*
optional
string <binary>
Responses
200 OK, this response indicates that you need to continue to upload the file.
201 CREATED, this response indicates that the upload is completed, and no further action is required.
400 BAD REQUEST, This response indicates that something is wrong about your request
401 UNAUTHORIZED, This response indicates that you are not authorized to access the expected resource
404 NOT FOUND, Resource not found
409 CONFLICT
413 PAYLOAD TOO LARGE, Payload Too Large
422 UNPROCESSABLE_ENTITY, this response indicates that the upload was completed but the sha256 hash comparison failed, hence the content has not been integrated.
451 UNAVAILABLE_FOR_LEGAL_REASONS, this response indicates that the upload was completed but the check of API tags is inconsistent.
default INTERNAL SERVER ERROR, An internal error occured
