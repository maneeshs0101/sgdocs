private boolean uploadDocumentContent(String documentId, File file, String token, RestTemplate restTemplate) throws Exception {
        //(auth)
        byte[] fileContent = java.nio.file.Files.readAllBytes(file.toPath());
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/pdf");
        headers.setContentLength(fileContent.length);
        headers.set(HttpHeaders.CONTENT_DISPOSITION,"binary;filename=\""+file.getName()+"\"");

        String uploadContentUrl = sgDocsV3BaseUrl + "/"+documentId+sgDocsV3ContentsPath;

        LOGGER.debug("Uploading content to  " + uploadContentUrl);

        HttpEntity<byte[]> requestEntity = new HttpEntity<>(fileContent, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                uploadContentUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if (response.getStatusCodeValue() == 201 || response.getStatusCodeValue() == 200) {
            LOGGER.info("Successfully uploaded document content for ID: " + documentId);
            return true;
        } else {
            LOGGER.error("Error uploading document content. Status " + response.getStatusCodeValue()
                    + ", Response: " + response.getBody());
            return false;
        }
    }
There's one problem 
byte[] fileContent = java.nio.file.Files.readAllBytes(file.toPath());
Here, while taking the path, the path is taken from the linux but i'm using windows,what can be done
