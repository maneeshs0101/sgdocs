
10:56:05.599 ERROR [SimpleAsyncTaskExecutor-1] c.s.custom.service.SGDocsProcessor
				[SGDocsProcessor.java:258] - Error uploading to SGM DOCS V3\home\niquat01\applis\niq\XELERATE\InvoicePDF\A407923010131445.pdf
java.nio.file.NoSuchFileException: \home\niquat01\applis\niq\XELERATE\InvoicePDF\A407923010131445.pdf
	at sun.nio.fs.WindowsException.translateToIOException(WindowsException.java:79)
	at sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:97)
	at sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:102)
	at sun.nio.fs.WindowsFileSystemProvider.newByteChannel(WindowsFileSystemProvider.java:230)
	at java.nio.file.Files.newByteChannel(Files.java:361)
	at java.nio.file.Files.newByteChannel(Files.java:407)
	at java.nio.file.Files.readAllBytes(Files.java:3152)
	at com.suntec.custom.service.SGDocsProcessor.uploadDocumentContent(SGDocsProcessor.java:280)

  package com.suntec.custom.service;

import java.io.File;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.*;

import org.apache.pdfbox.pdmodel.PDDocument;
import org.hibernate.validator.constraints.LuhnCheck;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.FileSystemResource;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.util.StringUtils;
import org.springframework.web.client.RestTemplate;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.suntec.custom.model.BillReports;
import com.suntec.custom.model.SGBillReports;
import com.suntec.custom.repository.BillReportRepository;
import com.suntec.custom.repository.SGBillReportRepository;

@Service
public class SGDocsProcessor implements ItemProcessor<SGBillReports, BillReports> {

    @Autowired
    private SGUtils sGUtils;

    @Autowired
    private BillReportRepository billReportRepository;

    @Autowired
    private SGBillReportRepository sGBillReportRepository;

//	  @Value("${sgdocs_scope}")
//	  private String sgdocs_scope;
//    @Value("${sgDocsUrl}")
//    private String sgDocsUrl;
//    @Value("${sgDocsGetpath}")
//    private String sgDocsGetpath;
//    @Value("${sgDocsUploadpath}")
//    private String sgDocsUploadpath;
    @Value("${sgDocsAuthor}")
    private String sgDocsAuthor;
//    @Value("${sgDocsType}")
//    private String sgDocsType;

    @Value("${sgDocsV3Scope}")
    private String sgDocsV3Scope;
    @Value("${sgDocsV3BaseUrl}")
    private String sgDocsV3BaseUrl;
    @Value("${sgDocsV3Workspaceid}")
    private String workspaceId;
    @Value("${sgDocsV3ModelId}")
    private String modelId;
    @Value("${sgDocsV3ContentsPath}")
    private String sgDocsV3ContentsPath;


    private static final Logger LOGGER = LoggerFactory.getLogger(SGDocsProcessor.class);

    @Override
    public BillReports process(SGBillReports sGBillReports) throws Exception {
        uploadDocument(sGBillReports);
        return null;
    }

//	@Transactional
//	private String uploadDocument(SGBillReports sGBillReports) throws Exception {
//		String sgDocsId = "";
//		int uploadStatus = 2;
//		String sgDocsErr = "";
//		int pageNos = 1;
//		if ("FC".equals(sGBillReports.getTechnicalFlag())) {
//			try {
//				String token = sGUtils.getToken(sgDocsV3Scope);
//				RestTemplate restTemplate = sGUtils.getRestTemplate(true);
//
//				HttpHeaders headers = new HttpHeaders();
//				headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
//				headers.set(HttpHeaders.CONTENT_TYPE, "multipart/form-data");
//				MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
//				File inFile = new File(sGBillReports.getReportPath() + "/" + sGBillReports.getFileName());
//				Map<String, String> sgDocsTypeMap = new ObjectMapper().readValue(sgDocsType,
//						new TypeReference<Map<String, Object>>() {
//						});
//				body.add("name", sGBillReports.getFileName());
//				body.add("type", sgDocsTypeMap.get(sGBillReports.getReportType()));
//				body.add("title", sGBillReports.getInvoicePointDescription());
//				body.add("author", sgDocsAuthor);
//				body.add("contentFormat", sGBillReports.getFileType());
//
//				Calendar c = Calendar.getInstance();
//				c.add(Calendar.YEAR, 10);
//				SimpleDateFormat sdf = new SimpleDateFormat("YYYY-MM-dd");
//				body.add("expirationDate", sdf.format(c.getTime()));
//				body.add("extendedProperties", "client_ID=" + sGBillReports.getClientId());
//				body.add("extendedProperties", "client_BIC=" + sGBillReports.getBicCode());
//				body.add("extendedProperties", "ref_document=" + sGBillReports.getFileName());
//				body.add("extendedProperties", "ref_document_maitre=" + sGBillReports.getBillNo() + ".pdf");
//				if ("QR".equalsIgnoreCase(sGBillReports.getReportType())
//						|| "POE".equalsIgnoreCase(sGBillReports.getReportType())) {
//					List<SGBillReports> sgBills = sGBillReportRepository
//							.getOriginalBillReport(sGBillReports.getBillNo());
//					if (sgBills != null && sgBills.size() > 0) {
//						SGBillReports sgBillReport = sgBills.get(0);
//						if (sgBillReport.getDebitAccount() != null && sgBillReport.getDebitAccount().length() == 16) {
//							body.add("extendedProperties",
//									"code_guichet_fact=" + sgBillReport.getDebitAccount().substring(0, 5));
//							body.add("extendedProperties",
//									"num_compte_fact=" + sgBillReport.getDebitAccount().substring(5, 16));
//						}
//
//						body.add("extendedProperties", "mode_paiement=" + sgBillReport.getPaymentMode());
//						if (sgBillReport.getTxnAccounts() != null) {
//							List<String> txnAccounts = Arrays.asList(sgBillReport.getTxnAccounts().split(","));
//
//							for (String txnAccount : txnAccounts) {
//								if (!StringUtils.isEmpty(txnAccount)
//										&& (txnAccount.startsWith("A_") || txnAccount.startsWith("B_")))
//									body.add("tags", txnAccount.substring(2));
//							}
//						}
//
//						if ("QR".equalsIgnoreCase(sGBillReports.getReportType())) {
//							PDDocument doc = PDDocument.load(inFile);
//							pageNos = doc.getNumberOfPages();
//							doc.close();
//							body.add("extendedProperties", "nb_pages=" + pageNos);
//						}
//					}
//
//				} else {
//					if (sGBillReports.getDebitAccount() != null && sGBillReports.getDebitAccount().length() == 16) {
//						body.add("extendedProperties",
//								"code_guichet_fact=" + sGBillReports.getDebitAccount().substring(0, 5));
//						body.add("extendedProperties",
//								"num_compte_fact=" + sGBillReports.getDebitAccount().substring(5, 16));
//					}
//					PDDocument doc = PDDocument.load(inFile);
//					pageNos = doc.getNumberOfPages();
//					doc.close();
//					body.add("extendedProperties", "nb_pages=" + (sGBillReports.getNoOfpages() == 0 ? 1 : pageNos));
//
//					body.add("extendedProperties", "mode_paiement=" + sGBillReports.getPaymentMode());
//					if (sGBillReports.getTxnAccounts() != null) {
//						List<String> txnAccounts = Arrays.asList(sGBillReports.getTxnAccounts().split(","));
//						for (String txnAccount : txnAccounts) {
//							if (!StringUtils.isEmpty(txnAccount)
//									&& (txnAccount.startsWith("A_") || txnAccount.startsWith("B_")))
//								body.add("tags", txnAccount.substring(2));
//						}
//					}
//				}
//
//				body.add("extendedProperties",
//						"mois_facture=" + sdf.format(sGBillReports.getBillDate()).substring(5, 7));
//				body.add("extendedProperties",
//						"annee_facture=" + sdf.format(sGBillReports.getBillDate()).substring(0, 4));
//				body.add("extendedProperties", "type_fact=" + sGBillReports.getTechnicalFlag());
//
//				LOGGER.debug("Uploading " + new ObjectMapper().writeValueAsString(body));
//				body.add("file", new FileSystemResource(inFile));
//				HttpEntity<MultiValueMap<String, Object>> requestEntity = new HttpEntity<MultiValueMap<String, Object>>(
//						body, headers);
//				String sgDocsUploadPath = sgDocsV3BaseUrl + sgDocsUploadpath;
//
//				ResponseEntity<String> response = restTemplate.exchange(sgDocsUploadPath, HttpMethod.POST,
//						requestEntity, String.class);
//
//				if (response.getStatusCodeValue() == 201) {
//					Map<String, Object> responseMap = new ObjectMapper().readValue(response.getBody(),
//							new TypeReference<Map<String, Object>>() {
//							});
//
//					sgDocsId = responseMap.get("id").toString();
//					uploadStatus = 1;
//					inFile.delete();
//
//				} else {
//					LOGGER.error("Error uploading to SG DOCS-" + response.getBody());
//					if (!StringUtils.isEmpty(response.getBody()))
//						sgDocsErr = response.getBody().substring(0, Math.min(response.getBody().length(), 200));
//				}
//			} catch (Exception ex) {
//				LOGGER.error("Error uploading to SG DOCS-" + ex.getMessage());
//				sgDocsErr = ex.getMessage().substring(0, Math.min(ex.getMessage().length(), 200));
//			}
//			LOGGER.debug("Response from SG DOCS-" + sgDocsId + "######uploadStatus-" + uploadStatus);
//		} else {
//			// Ignored invoices
//			uploadStatus = 3;
//		}
//		int updateCount = billReportRepository.updateUploadStatus(uploadStatus, sgDocsId, sGBillReports.getUniqueId(),
//				uploadStatus == 1 ? new Date() : null, sgDocsErr, pageNos);
//
//		LOGGER.debug("After updating bill_reports-" + updateCount);
//		return "";
//
//	}
//
//}

    /**
     * Two-Step process
     * 1.Validate document
     * 2.Get auth token
     * 3.Create document metadata in SGM Docs
     * 4.Upload document content file
     * 5.Update the db with upload status
     */
    @Transactional
    private String uploadDocument(SGBillReports sGBillReports) throws Exception {
        String sgDocsId = "";
        int uploadStatus = 2;
        String sgDocsErr = "";
        int pageNos = 1;

        if ("FC".equals(sGBillReports.getTechnicalFlag())) {
            try {
                String token = sGUtils.getToken(sgDocsV3Scope);
                RestTemplate restTemplate = sGUtils.getRestTemplate(true);

//            (page count needed for metadata before upload)
                File inFile = new File(sGBillReports.getReportPath() + "/" + sGBillReports.getFileName());


                // Create metadata
                String documentId = createDocumentMetadata(sGBillReports, token, pageNos, restTemplate);

                if (documentId != null && !documentId.isEmpty()) {
                    pageNos = calculatePageCount(inFile, sGBillReports.getReportType());

                    boolean contentUploaded = uploadDocumentContent(documentId, inFile, token, restTemplate);

                    if (contentUploaded) {
                        sgDocsId = documentId;
                        uploadStatus = 1;
                        inFile.delete();
                    } else {
                        sgDocsErr = "Failed to upload document content";
                    }
                } else {
                    sgDocsErr = "Failed to create document metadata";
                }
            } catch (Exception ex) {
                LOGGER.error("Error uploading to SGM DOCS V3{}", ex.getMessage(), ex);
                sgDocsErr = ex.getMessage().substring(0, Math.min(ex.getMessage().length(), 200));
            }
            LOGGER.debug("Response from SGM DOCS V3 - " + sgDocsId + "######uploadStatus-" + uploadStatus);
        } else {
            uploadStatus = 3;
        }

        int updateCount = billReportRepository.updateUploadStatus(
                uploadStatus,
                sgDocsId,
                sGBillReports.getUniqueId(),
                uploadStatus == 1 ? new Date() : null,
                sgDocsErr,
                pageNos
        );
        LOGGER.debug("After updating bill_reports: " + updateCount);
        return "";
    }

    private boolean uploadDocumentContent(String documentId, File file, String token, RestTemplate restTemplate) throws Exception {
        //(auth)
        byte[] fileContent=java.nio.file.Files.readAllBytes(file.toPath());
        if(fileContent==null || fileContent.length==0){
            LOGGER.error("Document content is empty for document ID: " + documentId);
            return false;
        }
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/pdf");

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

    private String createDocumentMetadata(SGBillReports sGBillReports,
                                          String token, int pageNos, RestTemplate restTemplate) throws Exception {
        //(auth)
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/json");

        //metadata object(custom fields)
        Map<String, Object> metadata = new LinkedHashMap<>();

        metadata.put("name", sGBillReports.getFileName());
        metadata.put("client_ID", sGBillReports.getClientId());
        metadata.put("client_BIC", sGBillReports.getBicCode());
        metadata.put("ref_document", sGBillReports.getFileName());
        metadata.put("ref_document_maitre", sGBillReports.getBillNo() + ".pdf");

        metadata.put("title", sGBillReports.getInvoicePointDescription());
        metadata.put("author", sgDocsAuthor);

        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        metadata.put("mois_facture", sdf.format(sGBillReports.getBillDate()).substring(5, 7));
        metadata.put("annee_facture", sdf.format(sGBillReports.getBillDate()).substring(0, 4));
        metadata.put("type_fact", sGBillReports.getTechnicalFlag());


        //determine source report
        //for "QR" and "POE" report types, get metadata from original bill
  /*
        v2
 if (QR or POE)
   Get original bill
   Extract account
   Add payment mode
   Extract tags
   Calculate pages
 else
   Extract account
   Add payment mode
   Extract tags
   Calculate pages
   */
/*
    v3
 sourceReport = (QR/POE)? originalBill : currentReport
 addAccountMetadata(source)
 add payment mode
 extractTags(source)

 (execute once)
 */

        SGBillReports sourceReport = sGBillReports;
        if ("QR".equalsIgnoreCase(sGBillReports.getReportType()) ||
                "POE".equalsIgnoreCase(sGBillReports.getReportType())) {

            //Fetch the original bill report
            List<SGBillReports> sgBills = sGBillReportRepository.getOriginalBillReport(sGBillReports.getBillNo());
            if (sgBills != null && sgBills.size() > 0) {
                sourceReport = sgBills.get(0);
            }
        }

        addAccountMetadata(sourceReport, metadata);

        metadata.put("mode_paiement", sourceReport.getPaymentMode());
        metadata.put("nb_pages", pageNos);

        String tags = extractTags(sourceReport);
        if (tags != null && !tags.isEmpty()) {
            metadata.put("tags", tags);
        }

        //calculating document expiration/destruction date (10 years from now)
        Calendar c = Calendar.getInstance();
        c.add(Calendar.YEAR, 10);
        String destroyDate = sdf.format(c.getTime());

        //complete req body
        Map<String, Object> requestBody = new LinkedHashMap<>();

        requestBody.put("modelId", modelId);
        //Add all metadata
        requestBody.put("metadata", metadata);

        //doc lifecycle and security settings
        requestBody.put("destroyDate", destroyDate);
        requestBody.put("confidentialityLevel","C2");

        //Convert to JSON and log
        String requestJson =  new ObjectMapper().writeValueAsString(requestBody);
        LOGGER.debug("Creating document metadata: " + requestJson);


        HttpEntity<String> requestEntity = new HttpEntity<>(requestJson, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                sgDocsV3BaseUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if(response.getStatusCodeValue()==201||response.getStatusCodeValue()==200) {
            Map<String, Object> responseMap = new ObjectMapper().readValue(response.getBody(),
                    new TypeReference<Map<String, Object>>() {
                    });

            Map<String, Object> docVerIdentifier= (Map<String, Object>) responseMap.get("documentVersionIdentifier");

            String documentId =docVerIdentifier.get("documentId").toString();
            LOGGER.info("Succefully created document metadata with ID: " + documentId);
            return documentId;
        } else{
            LOGGER.error("Error creating document metadata. Status " + response.getStatusCodeValue()
            +", Response: "+response.getBody());
            return null;
        }
    }

    private String extractTags(SGBillReports report) {
        if (report.getTxnAccounts() != null) {
            List<String> tagList = new ArrayList<>();
            List<String> txnAccounts = Arrays.asList(report.getTxnAccounts().split(","));

            for (String txnAccount : txnAccounts) {
                if (!StringUtils.isEmpty(txnAccount) &&
                        (txnAccount.startsWith("A_") || txnAccount.startsWith("B_"))) {
                    tagList.add(txnAccount.substring(2));
                }
            }
            return tagList.isEmpty() ? null : String.join(",", tagList);
        }
        return null;
    }

    private void addAccountMetadata(SGBillReports report, Map<String, Object> metadata) {
        if(report.getDebitAccount()!=null && report.getDebitAccount().length()==16){
            metadata.put("code_guichet_fact",report.getDebitAccount().substring(0,5));
            metadata.put("num_compte_fact",report.getDebitAccount().substring(5,16));
        }
    }

    private int calculatePageCount(File file, String reportType) {
        try{
            PDDocument doc=PDDocument.load(file);
            int pageCount=doc.getNumberOfPages();
            doc.close();
            LOGGER.debug("Page count for reportType "+reportType+" is "+pageCount);
            return  pageCount;
        } catch (Exception e) {
            LOGGER.error("Error calculating page count - "+e.getMessage());
            return 1;
        }
    }

}
