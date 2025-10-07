Create a document
PREREQUISITE: In order to create a document, the creator of the document needs to be one of the authorized contributors.

In the first step, we create a new document by uploading metadata, then in the second step we add a text file to it.

http
First step: POST metadata
POST /api/v3/workspaces/[InsertWorkspaceId]/documents/ HTTP/1.1
Host: api.sgdocs.prd.euw.gbis.sg-azure.com
content-type: application/json
Authorization: Bearer [InsertTokenHere]
Content-Length: 131

{
    "modelId": "[InsertModelId]",
    "metadata": {
        "documentName": "My first document"
    }
}
HTTP
Second step: POST content
POST /api/v3/workspaces/[InsertWorkspaceId]/documents/[InsertDocumentId]/contents HTTP/1.1
Host: api.sgdocs.prd.euw.gbis.sg-azure.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: binary;filename="[InsertFileName]"
Authorization: Bearer [InsertTokenHere]
Content-Length: 250

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="data"; filename="[InsertFilePath]"
Content-Type: text/plain

(data)
------WebKitFormBoundary7MA4YWxkTrZu0gW--
