package com.suntec.custom.service;

import java.io.File;
import java.io.IOException;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.text.SimpleDateFormat;
import java.util.*;

import org.apache.commons.io.FileUtils;
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

    @Value("${sgDocsAuthor}")
    private String sgDocsAuthor;
    @Value("${sgDocsV3Scope}")
    private String sgDocsV3Scope;
    @Value("${sgDocsPV3BaseUrl}")
    private String sgDocsV3BaseUrl;
    @Value("${sgDocsPV3Workspaceid}")
    private String workspaceId;
    @Value("${sgDocsPV3ModelId}")
    private String modelId;
    @Value("${sgDocsV3ContentsPath}")
    private String sgDocsV3ContentsPath;


    private static final Logger LOGGER = LoggerFactory.getLogger(SGDocsProcessor.class);

    @Override
    public BillReports process(SGBillReports sGBillReports) throws Exception
    {
        uploadDocument(sGBillReports);
        return null;
    }

    @Transactional
    private String uploadDocument(SGBillReports sGBillReports) throws Exception
    {
        String sgDocsId = "";
        int uploadStatus = 2;
        String sgDocsErr = "";
        int pageNos = 1;

        if ("FC".equals(sGBillReports.getTechnicalFlag()))
        {
            try
            {
                String token = sGUtils.getToken(sgDocsV3Scope);
                RestTemplate restTemplate = sGUtils.getRestTemplate(true);

                File inFile = new File(sGBillReports.getReportPath()+"/"+ sGBillReports.getFileName());

                String documentId = createDocumentMetadata(sGBillReports, token, pageNos, restTemplate);

                if (documentId != null && !documentId.isEmpty())
                {
                    pageNos = calculatePageCount(inFile, sGBillReports.getReportType());

                    boolean contentUploaded = uploadDocumentContent(documentId, inFile, token, restTemplate);

                    if (contentUploaded)
                    {
                        sgDocsId = documentId;
                        uploadStatus = 1;
                        inFile.delete();
                    }
                    else
                    {
                        sgDocsErr = "Failed to upload document content";
                    }
                }
                else
                {
                    sgDocsErr = "Failed to create document metadata";
                }
            } catch (Exception ex)
            {
                LOGGER.error("Error uploading to SGM DOCS V3{}", ex.getMessage(), ex);
                sgDocsErr = ex.getMessage().substring(0, Math.min(ex.getMessage().length(), 200));
            }

            LOGGER.debug("Response from SGM DOCS V3 - " + sgDocsId + "######uploadStatus-" + uploadStatus);
        }
        else
        {
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

    private boolean uploadDocumentContent(String documentId, File file, String token, RestTemplate restTemplate) throws Exception
    {
        FileSystemResource fileResource = new FileSystemResource(file);
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/pdf");
        headers.setContentLength(fileResource.contentLength());
        headers.set(HttpHeaders.CONTENT_DISPOSITION,"binary;filename=\""+file.getName()+"\"");

        String uploadContentUrl = sgDocsV3BaseUrl + "/"+documentId+sgDocsV3ContentsPath;

        LOGGER.debug("Uploading content to  " + uploadContentUrl);

        HttpEntity<FileSystemResource> requestEntity = new HttpEntity<>(fileResource, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                uploadContentUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if (response.getStatusCodeValue() == 201 || response.getStatusCodeValue() == 200)
        {
            LOGGER.info("Successfully uploaded document content for ID: " + documentId);
            return true;
        }
        else
        {
            LOGGER.error("Error uploading document content. Status " + response.getStatusCodeValue()
                    + ", Response: " + response.getBody());
            return false;
        }
    }

    private String createDocumentMetadata(SGBillReports sGBillReports,
                                          String token, int pageNos, RestTemplate restTemplate) throws Exception
    {
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/json");

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

        SGBillReports sourceReport = sGBillReports;
        if ("QR".equalsIgnoreCase(sGBillReports.getReportType()) ||
                "POE".equalsIgnoreCase(sGBillReports.getReportType()))
        {
            List<SGBillReports> sgBills = sGBillReportRepository.getOriginalBillReport(sGBillReports.getBillNo());
            if (sgBills != null && sgBills.size() > 0) {
                sourceReport = sgBills.get(0);
            }
        }

        addAccountMetadata(sourceReport, metadata);

        metadata.put("mode_paiement", sourceReport.getPaymentMode());
        metadata.put("nb_pages", pageNos);

        String tags = extractTags(sourceReport);
        if (tags != null && !tags.isEmpty())
        {
            metadata.put("tags", tags);
        }

        Calendar c = Calendar.getInstance();
        c.add(Calendar.YEAR, 10);
        String destroyDate = sdf.format(c.getTime());

        Map<String, Object> requestBody = new LinkedHashMap<>();

        requestBody.put("modelId", modelId);
        requestBody.put("metadata", metadata);
        requestBody.put("destroyDate", destroyDate);
        requestBody.put("confidentialityLevel","C2");

        String requestJson =  new ObjectMapper().writeValueAsString(requestBody);
        LOGGER.debug("Creating document metadata: " + requestJson);

        HttpEntity<String> requestEntity = new HttpEntity<>(requestJson, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                sgDocsV3BaseUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if(response.getStatusCodeValue()==201||response.getStatusCodeValue()==200)
        {
            Map<String, Object> responseMap = new ObjectMapper().readValue(response.getBody(),
                    new TypeReference<Map<String, Object>>() {
                    });

            Map<String, Object> docVerIdentifier= (Map<String, Object>) responseMap.get("documentVersionIdentifier");

            String documentId =docVerIdentifier.get("documentId").toString();
            LOGGER.info("Succefully created document metadata with ID: " + documentId);
            return documentId;
        }
        else
        {
            LOGGER.error("Error creating document metadata. Status " + response.getStatusCodeValue()
            +", Response: "+response.getBody());
            return null;
        }
    }

    private String extractTags(SGBillReports report)
    {
        if (report.getTxnAccounts() != null)
        {
            List<String> tagList = new ArrayList<>();
            List<String> txnAccounts = Arrays.asList(report.getTxnAccounts().split(","));

            for (String txnAccount : txnAccounts)
            {
                if (!StringUtils.isEmpty(txnAccount) &&
                        (txnAccount.startsWith("A_") || txnAccount.startsWith("B_")))
                {
                    tagList.add(txnAccount.substring(2));
                }
            }
            return tagList.isEmpty() ? null : String.join(",", tagList);
        }
        return null;
    }

    private void addAccountMetadata(SGBillReports report, Map<String, Object> metadata)
    {
        if(report.getDebitAccount()!=null && report.getDebitAccount().length()==16)
        {
            metadata.put("code_guichet_fact",report.getDebitAccount().substring(0,5));
            metadata.put("num_compte_fact",report.getDebitAccount().substring(5,16));
        }
    }

    private int calculatePageCount(File file, String reportType)
    {
        try
        {
            PDDocument doc=PDDocument.load(file);
            int pageCount=doc.getNumberOfPages();
            doc.close();
            LOGGER.debug("Page count for reportType "+reportType+" is "+pageCount);
            return  pageCount;
        }
        catch (Exception e)
        {
            LOGGER.error("Error calculating page count - "+e.getMessage());
            return 1;
        }
    }

}
package com.suntec.custom.service;

import java.io.File;
import java.io.IOException;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.text.SimpleDateFormat;
import java.util.*;

import org.apache.commons.io.FileUtils;
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

    @Value("${sgDocsAuthor}")
    private String sgDocsAuthor;
    @Value("${sgDocsV3Scope}")
    private String sgDocsV3Scope;
    @Value("${sgDocsPV3BaseUrl}")
    private String sgDocsV3BaseUrl;
    @Value("${sgDocsPV3Workspaceid}")
    private String workspaceId;
    @Value("${sgDocsPV3ModelId}")
    private String modelId;
    @Value("${sgDocsV3ContentsPath}")
    private String sgDocsV3ContentsPath;


    private static final Logger LOGGER = LoggerFactory.getLogger(SGDocsProcessor.class);

    @Override
    public BillReports process(SGBillReports sGBillReports) throws Exception
    {
        uploadDocument(sGBillReports);
        return null;
    }

    @Transactional
    private String uploadDocument(SGBillReports sGBillReports) throws Exception
    {
        String sgDocsId = "";
        int uploadStatus = 2;
        String sgDocsErr = "";
        int pageNos = 1;

        if ("FC".equals(sGBillReports.getTechnicalFlag()))
        {
            try
            {
                String token = sGUtils.getToken(sgDocsV3Scope);
                RestTemplate restTemplate = sGUtils.getRestTemplate(true);

                File inFile = new File(sGBillReports.getReportPath()+"/"+ sGBillReports.getFileName());

                String documentId = createDocumentMetadata(sGBillReports, token, pageNos, restTemplate);

                if (documentId != null && !documentId.isEmpty())
                {
                    pageNos = calculatePageCount(inFile, sGBillReports.getReportType());

                    boolean contentUploaded = uploadDocumentContent(documentId, inFile, token, restTemplate);

                    if (contentUploaded)
                    {
                        sgDocsId = documentId;
                        uploadStatus = 1;
                        inFile.delete();
                    }
                    else
                    {
                        sgDocsErr = "Failed to upload document content";
                    }
                }
                else
                {
                    sgDocsErr = "Failed to create document metadata";
                }
            } catch (Exception ex)
            {
                LOGGER.error("Error uploading to SGM DOCS V3{}", ex.getMessage(), ex);
                sgDocsErr = ex.getMessage().substring(0, Math.min(ex.getMessage().length(), 200));
            }

            LOGGER.debug("Response from SGM DOCS V3 - " + sgDocsId + "######uploadStatus-" + uploadStatus);
        }
        else
        {
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

    private boolean uploadDocumentContent(String documentId, File file, String token, RestTemplate restTemplate) throws Exception
    {
        FileSystemResource fileResource = new FileSystemResource(file);
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/pdf");
        headers.setContentLength(fileResource.contentLength());
        headers.set(HttpHeaders.CONTENT_DISPOSITION,"binary;filename=\""+file.getName()+"\"");

        String uploadContentUrl = sgDocsV3BaseUrl + "/"+documentId+sgDocsV3ContentsPath;

        LOGGER.debug("Uploading content to  " + uploadContentUrl);

        HttpEntity<FileSystemResource> requestEntity = new HttpEntity<>(fileResource, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                uploadContentUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if (response.getStatusCodeValue() == 201 || response.getStatusCodeValue() == 200)
        {
            LOGGER.info("Successfully uploaded document content for ID: " + documentId);
            return true;
        }
        else
        {
            LOGGER.error("Error uploading document content. Status " + response.getStatusCodeValue()
                    + ", Response: " + response.getBody());
            return false;
        }
    }

    private String createDocumentMetadata(SGBillReports sGBillReports,
                                          String token, int pageNos, RestTemplate restTemplate) throws Exception
    {
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/json");

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

        SGBillReports sourceReport = sGBillReports;
        if ("QR".equalsIgnoreCase(sGBillReports.getReportType()) ||
                "POE".equalsIgnoreCase(sGBillReports.getReportType()))
        {
            List<SGBillReports> sgBills = sGBillReportRepository.getOriginalBillReport(sGBillReports.getBillNo());
            if (sgBills != null && sgBills.size() > 0) {
                sourceReport = sgBills.get(0);
            }
        }

        addAccountMetadata(sourceReport, metadata);

        metadata.put("mode_paiement", sourceReport.getPaymentMode());
        metadata.put("nb_pages", pageNos);

        String tags = extractTags(sourceReport);
        if (tags != null && !tags.isEmpty())
        {
            metadata.put("tags", tags);
        }

        Calendar c = Calendar.getInstance();
        c.add(Calendar.YEAR, 10);
        String destroyDate = sdf.format(c.getTime());

        Map<String, Object> requestBody = new LinkedHashMap<>();

        requestBody.put("modelId", modelId);
        requestBody.put("metadata", metadata);
        requestBody.put("destroyDate", destroyDate);
        requestBody.put("confidentialityLevel","C2");

        String requestJson =  new ObjectMapper().writeValueAsString(requestBody);
        LOGGER.debug("Creating document metadata: " + requestJson);

        HttpEntity<String> requestEntity = new HttpEntity<>(requestJson, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                sgDocsV3BaseUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if(response.getStatusCodeValue()==201||response.getStatusCodeValue()==200)
        {
            Map<String, Object> responseMap = new ObjectMapper().readValue(response.getBody(),
                    new TypeReference<Map<String, Object>>() {
                    });

            Map<String, Object> docVerIdentifier= (Map<String, Object>) responseMap.get("documentVersionIdentifier");

            String documentId =docVerIdentifier.get("documentId").toString();
            LOGGER.info("Succefully created document metadata with ID: " + documentId);
            return documentId;
        }
        else
        {
            LOGGER.error("Error creating document metadata. Status " + response.getStatusCodeValue()
            +", Response: "+response.getBody());
            return null;
        }
    }

    private String extractTags(SGBillReports report)
    {
        if (report.getTxnAccounts() != null)
        {
            List<String> tagList = new ArrayList<>();
            List<String> txnAccounts = Arrays.asList(report.getTxnAccounts().split(","));

            for (String txnAccount : txnAccounts)
            {
                if (!StringUtils.isEmpty(txnAccount) &&
                        (txnAccount.startsWith("A_") || txnAccount.startsWith("B_")))
                {
                    tagList.add(txnAccount.substring(2));
                }
            }
            return tagList.isEmpty() ? null : String.join(",", tagList);
        }
        return null;
    }

    private void addAccountMetadata(SGBillReports report, Map<String, Object> metadata)
    {
        if(report.getDebitAccount()!=null && report.getDebitAccount().length()==16)
        {
            metadata.put("code_guichet_fact",report.getDebitAccount().substring(0,5));
            metadata.put("num_compte_fact",report.getDebitAccount().substring(5,16));
        }
    }

    private int calculatePageCount(File file, String reportType)
    {
        try
        {
            PDDocument doc=PDDocument.load(file);
            int pageCount=doc.getNumberOfPages();
            doc.close();
            LOGGER.debug("Page count for reportType "+reportType+" is "+pageCount);
            return  pageCount;
        }
        catch (Exception e)
        {
            LOGGER.error("Error calculating page count - "+e.getMessage());
            return 1;
        }
    }

}
package com.suntec.custom.service;

import java.io.File;
import java.io.IOException;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.text.SimpleDateFormat;
import java.util.*;

import org.apache.commons.io.FileUtils;
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

    @Value("${sgDocsAuthor}")
    private String sgDocsAuthor;
    @Value("${sgDocsV3Scope}")
    private String sgDocsV3Scope;
    @Value("${sgDocsPV3BaseUrl}")
    private String sgDocsV3BaseUrl;
    @Value("${sgDocsPV3Workspaceid}")
    private String workspaceId;
    @Value("${sgDocsPV3ModelId}")
    private String modelId;
    @Value("${sgDocsV3ContentsPath}")
    private String sgDocsV3ContentsPath;


    private static final Logger LOGGER = LoggerFactory.getLogger(SGDocsProcessor.class);

    @Override
    public BillReports process(SGBillReports sGBillReports) throws Exception
    {
        uploadDocument(sGBillReports);
        return null;
    }

    @Transactional
    private String uploadDocument(SGBillReports sGBillReports) throws Exception
    {
        String sgDocsId = "";
        int uploadStatus = 2;
        String sgDocsErr = "";
        int pageNos = 1;

        if ("FC".equals(sGBillReports.getTechnicalFlag()))
        {
            try
            {
                String token = sGUtils.getToken(sgDocsV3Scope);
                RestTemplate restTemplate = sGUtils.getRestTemplate(true);

                File inFile = new File(sGBillReports.getReportPath()+"/"+ sGBillReports.getFileName());

                String documentId = createDocumentMetadata(sGBillReports, token, pageNos, restTemplate);

                if (documentId != null && !documentId.isEmpty())
                {
                    pageNos = calculatePageCount(inFile, sGBillReports.getReportType());

                    boolean contentUploaded = uploadDocumentContent(documentId, inFile, token, restTemplate);

                    if (contentUploaded)
                    {
                        sgDocsId = documentId;
                        uploadStatus = 1;
                        inFile.delete();
                    }
                    else
                    {
                        sgDocsErr = "Failed to upload document content";
                    }
                }
                else
                {
                    sgDocsErr = "Failed to create document metadata";
                }
            } catch (Exception ex)
            {
                LOGGER.error("Error uploading to SGM DOCS V3{}", ex.getMessage(), ex);
                sgDocsErr = ex.getMessage().substring(0, Math.min(ex.getMessage().length(), 200));
            }

            LOGGER.debug("Response from SGM DOCS V3 - " + sgDocsId + "######uploadStatus-" + uploadStatus);
        }
        else
        {
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

    private boolean uploadDocumentContent(String documentId, File file, String token, RestTemplate restTemplate) throws Exception
    {
        FileSystemResource fileResource = new FileSystemResource(file);
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/pdf");
        headers.setContentLength(fileResource.contentLength());
        headers.set(HttpHeaders.CONTENT_DISPOSITION,"binary;filename=\""+file.getName()+"\"");

        String uploadContentUrl = sgDocsV3BaseUrl + "/"+documentId+sgDocsV3ContentsPath;

        LOGGER.debug("Uploading content to  " + uploadContentUrl);

        HttpEntity<FileSystemResource> requestEntity = new HttpEntity<>(fileResource, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                uploadContentUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if (response.getStatusCodeValue() == 201 || response.getStatusCodeValue() == 200)
        {
            LOGGER.info("Successfully uploaded document content for ID: " + documentId);
            return true;
        }
        else
        {
            LOGGER.error("Error uploading document content. Status " + response.getStatusCodeValue()
                    + ", Response: " + response.getBody());
            return false;
        }
    }

    private String createDocumentMetadata(SGBillReports sGBillReports,
                                          String token, int pageNos, RestTemplate restTemplate) throws Exception
    {
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        headers.set(HttpHeaders.CONTENT_TYPE, "application/json");

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

        SGBillReports sourceReport = sGBillReports;
        if ("QR".equalsIgnoreCase(sGBillReports.getReportType()) ||
                "POE".equalsIgnoreCase(sGBillReports.getReportType()))
        {
            List<SGBillReports> sgBills = sGBillReportRepository.getOriginalBillReport(sGBillReports.getBillNo());
            if (sgBills != null && sgBills.size() > 0) {
                sourceReport = sgBills.get(0);
            }
        }

        addAccountMetadata(sourceReport, metadata);

        metadata.put("mode_paiement", sourceReport.getPaymentMode());
        metadata.put("nb_pages", pageNos);

        String tags = extractTags(sourceReport);
        if (tags != null && !tags.isEmpty())
        {
            metadata.put("tags", tags);
        }

        Calendar c = Calendar.getInstance();
        c.add(Calendar.YEAR, 10);
        String destroyDate = sdf.format(c.getTime());

        Map<String, Object> requestBody = new LinkedHashMap<>();

        requestBody.put("modelId", modelId);
        requestBody.put("metadata", metadata);
        requestBody.put("destroyDate", destroyDate);
        requestBody.put("confidentialityLevel","C2");

        String requestJson =  new ObjectMapper().writeValueAsString(requestBody);
        LOGGER.debug("Creating document metadata: " + requestJson);

        HttpEntity<String> requestEntity = new HttpEntity<>(requestJson, headers);

        ResponseEntity<String> response = restTemplate.exchange(
                sgDocsV3BaseUrl,
                HttpMethod.POST,
                requestEntity,
                String.class
        );

        if(response.getStatusCodeValue()==201||response.getStatusCodeValue()==200)
        {
            Map<String, Object> responseMap = new ObjectMapper().readValue(response.getBody(),
                    new TypeReference<Map<String, Object>>() {
                    });

            Map<String, Object> docVerIdentifier= (Map<String, Object>) responseMap.get("documentVersionIdentifier");

            String documentId =docVerIdentifier.get("documentId").toString();
            LOGGER.info("Succefully created document metadata with ID: " + documentId);
            return documentId;
        }
        else
        {
            LOGGER.error("Error creating document metadata. Status " + response.getStatusCodeValue()
            +", Response: "+response.getBody());
            return null;
        }
    }

    private String extractTags(SGBillReports report)
    {
        if (report.getTxnAccounts() != null)
        {
            List<String> tagList = new ArrayList<>();
            List<String> txnAccounts = Arrays.asList(report.getTxnAccounts().split(","));

            for (String txnAccount : txnAccounts)
            {
                if (!StringUtils.isEmpty(txnAccount) &&
                        (txnAccount.startsWith("A_") || txnAccount.startsWith("B_")))
                {
                    tagList.add(txnAccount.substring(2));
                }
            }
            return tagList.isEmpty() ? null : String.join(",", tagList);
        }
        return null;
    }

    private void addAccountMetadata(SGBillReports report, Map<String, Object> metadata)
    {
        if(report.getDebitAccount()!=null && report.getDebitAccount().length()==16)
        {
            metadata.put("code_guichet_fact",report.getDebitAccount().substring(0,5));
            metadata.put("num_compte_fact",report.getDebitAccount().substring(5,16));
        }
    }

    private int calculatePageCount(File file, String reportType)
    {
        try
        {
            PDDocument doc=PDDocument.load(file);
            int pageCount=doc.getNumberOfPages();
            doc.close();
            LOGGER.debug("Page count for reportType "+reportType+" is "+pageCount);
            return  pageCount;
        }
        catch (Exception e)
        {
            LOGGER.error("Error calculating page count - "+e.getMessage());
            return 1;
        }
    }

}
@A724878 commented on this pull request.
•	Have to put tests
•	have to validate the response from the API (Ex : doc ID, ...) not only the response code
