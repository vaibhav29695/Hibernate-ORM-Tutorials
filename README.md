=====
package com.epay.admin.portal.validator;

import com.sbi.epay.logging.utility.LoggerFactoryUtility;
import com.sbi.epay.logging.utility.LoggerUtility;
import org.springframework.web.multipart.MultipartFile;

import java.util.ArrayList;
import java.util.List;

public class ChargebackFileUploadValidator extends BaseValidator {

    private static final String FIELD_CARD_ISSUER = "cardIssuer";
    private static final String FIELD_CB_STAGE = "cbStage";
    private static final String FIELD_FILE = "file";

    private static final String INVALID_FILE_ERROR_CODE = "INVALID_FILE";
    private static final String INVALID_FILE_ERROR_MESSAGE =
            "Invalid file. Only XLSX, XLS and CSV files are allowed.";

    private static final List<String> ALLOWED_FILE_EXTENSIONS =
            List.of("xlsx", "xls", "csv");

    private final LoggerUtility logger =
            LoggerFactoryUtility.getLogger(this.getClass());

    /**
     * Validates Chargeback file upload request.
     *
     * @param cardIssuer Card issuer
     * @param cbStage    Chargeback stage
     * @param file       Chargeback upload file
     */
    public void validateChargebackFileUpload(
            String cardIssuer,
            String cbStage,
            MultipartFile file) {

        errorDtoList = new ArrayList<>();

        logger.info("validateChargebackFileUpload - validation started");

        // Mandatory validations
        checkMandatoryField(cardIssuer, FIELD_CARD_ISSUER);
        checkMandatoryField(cbStage, FIELD_CB_STAGE);
        checkMandatoryField(file, FIELD_FILE);

        // File validation
        if (file != null && !file.isEmpty()) {
            validateFileExtension(file);
        }

        // Throw validation errors if any
        throwIfErrors();
    }

    /**
     * Validates uploaded file extension.
     */
    private void validateFileExtension(MultipartFile file) {

        String originalFileName = file.getOriginalFilename();

        if (originalFileName == null || originalFileName.isBlank()) {
            addError(
                    FIELD_FILE,
                    INVALID_FILE_ERROR_CODE,
                    "File name is required."
            );
            return;
        }

        String extension = getFileExtension(originalFileName);

        if (!ALLOWED_FILE_EXTENSIONS.contains(extension.toLowerCase())) {
            addError(
                    FIELD_FILE,
                    INVALID_FILE_ERROR_CODE,
                    INVALID_FILE_ERROR_MESSAGE
            );
        }
    }

    /**
     * Extracts file extension from file name.
     */
    private String getFileExtension(String fileName) {

        int lastDotIndex = fileName.lastIndexOf('.');

        if (lastDotIndex == -1 || lastDotIndex == fileName.length() - 1) {
            return "";
        }

        return fileName.substring(lastDotIndex + 1);
    }
}

//////
package com.epay.admin.portal.externalservice;

import com.epay.admin.portal.config.S3Config;
import com.epay.admin.portal.exceptions.AdminPortalException;
import com.epay.admin.portal.externalservice.response.simulator.RBIMalwareDownloadResponse;
import com.epay.admin.portal.util.AdminPortalUtil;
import com.epay.admin.portal.util.ErrorConstants;
import com.sbi.epay.logging.utility.LoggerFactoryUtility;
import com.sbi.epay.logging.utility.LoggerUtility;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.apache.commons.lang3.ObjectUtils;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import software.amazon.awssdk.core.ResponseInputStream;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.text.MessageFormat;
import java.util.ArrayList;
import java.util.List;

import static com.epay.admin.portal.util.AdminPortalConstants.S3_FILE_NAME_FORMAT;

/**
 * Class Name: S3Service
 * Description: The S3Service class provides functionalities to interact with AWS S3 for uploading, downloading, and listing files.
 * It supports uploading files as a File, byte array, or MultipartFile, and handles the S3 client operations like put, get, and list objects.
 * It also provides error handling, logging, and custom exception throwing in case of S3 operation failures.
 * Author: V1017794 - Hariom Kumar
 * Copyright (c) 2025 [State Bank of India]
 * All rights reserved
 * Version: 1.0
 */

@Service
@Primary
@RequiredArgsConstructor
public class S3Service implements FileService {
    private final LoggerUtility log = LoggerFactoryUtility.getLogger(this.getClass());
    private final S3Config s3Config;
    private final S3Client s3Client;

    /**
     * Uploads a file to S3 using a File object.
     *
     * @param file the file to be uploaded
     * @return the S3 key for the uploaded file
     */
    @Override
    public String uploadFile(File file) {
        String key = System.currentTimeMillis() + "-" + file.getName();
        try {
            log.info("Uploading file to S3: {}", key);
            PutObjectRequest objectRequest = PutObjectRequest.builder().bucket(s3Config.getBucket()).key(file.getName()).build();
            s3Client.putObject(objectRequest, RequestBody.fromFile(file));
            log.info("File uploaded successfully: {}", key);
            return key;
        } catch (S3Exception e) {
            log.error(ErrorConstants.S3_UPLOAD_FAILED, key, e.getMessage());
            throw new AdminPortalException(ErrorConstants.GENERIC_ERROR_CODE, MessageFormat.format(ErrorConstants.S3_UPLOAD_FAILED, key, e.getMessage()));
        }
    }

    public String uploadMultipartFile(MultipartFile multipartFile) {

        String key = System.currentTimeMillis()
                + "_" + multipartFile.getOriginalFilename();

        try {

            log.info("Uploading file to S3: {}", key);

            PutObjectRequest objectRequest =
                    PutObjectRequest.builder()
                            .bucket(s3Config.getBucket())
                            .key(key)
                            .build();

//            s3Client.putObject(
//                    objectRequest,
//                    RequestBody.fromInputStream(multipartFile.getInputStream(),multipartFile.getSize())
//            );

            log.info("File uploaded successfully to S3: {}", key);

            return key;

        } catch (Exception e) {

            log.error("S3 upload failed: {}", e.getMessage(), e);

            throw new AdminPortalException(
                    ErrorConstants.GENERIC_ERROR_CODE,
                    MessageFormat.format(
                            ErrorConstants.S3_UPLOAD_FAILED,
                            key
                    )
            );
        }
    }


    /**
     * Uploads a file to S3 using byte array content.
     *
     * @param mifId              mifId of the document
     * @param documentType       documentType of the file
     * @param rbiMalwareResponse RBIMalwareDownloadResponse - the file content as a byte array and file content type
     * @return the S3 key for the uploaded file
     */
    @Override
    public String uploadFile(String mifId, String documentType, RBIMalwareDownloadResponse rbiMalwareResponse) {
        String extension = getExtensionFromMime(rbiMalwareResponse.getContentType());
        String key = String.format(S3_FILE_NAME_FORMAT, mifId, documentType, System.currentTimeMillis(), extension);
        try {
            log.info("Uploading byte array file to S3: {}", key);
            PutObjectRequest objectRequest = PutObjectRequest.builder().bucket(s3Config.getBucket()).key(key).build();
            s3Client.putObject(objectRequest, RequestBody.fromBytes(rbiMalwareResponse.getFileContent()));
            log.info("File uploaded successfully: {}", key);
            return key;
        } catch (S3Exception e) {
            log.error(ErrorConstants.S3_UPLOAD_FAILED, key, e.getMessage());
            throw new AdminPortalException(ErrorConstants.GENERIC_ERROR_CODE, MessageFormat.format(ErrorConstants.S3_UPLOAD_FAILED, key, e.getMessage()));
        }
    }

    /**
     * Uploads a file to S3 using a MultipartFile.
     *
     * @param file the MultipartFile to upload
     * @return the S3 key for the uploaded file
     */
    @Override
    public String uploadFile(MultipartFile file) {
        String key = System.currentTimeMillis() + "-" + file.getName();
        try {
            log.info("Uploading MultipartFile to S3: {}", key);
            PutObjectRequest objectRequest = PutObjectRequest.builder().bucket(s3Config.getBucket()).key(key).contentType(file.getContentType()).contentLength(file.getSize()).build();
            PutObjectResponse putObjectResponse = s3Client.putObject(objectRequest, RequestBody.fromInputStream(file.getInputStream(), file.getSize()));
            log.info("File uploaded successfully: {}", key);
            return key;
        } catch (S3Exception | IOException e) {
            log.error(ErrorConstants.S3_UPLOAD_FAILED, key, e.getMessage());
            throw new AdminPortalException(ErrorConstants.GENERIC_ERROR_CODE, e.getMessage());
        }
    }

    /**
     * Downloads a file from S3 and writes it to the provided HttpServletResponse.
     *
     * @param response the HTTP response object where the file will be written
     * @param fileName the name of the file to be downloaded
     */

    @Override
    public void downloadFile(HttpServletResponse response, String fileName) {
        try {
            log.info("Downloading file from S3: {}", fileName);
            GetObjectRequest getObjectRequest = GetObjectRequest.builder().bucket(s3Config.getBucket()).key(fileName).build();
            ResponseInputStream<GetObjectResponse> object = s3Client.getObject(getObjectRequest);
            AdminPortalUtil.setHeader(response, fileName, object.response().contentLength());
            try (InputStream in = object; OutputStream out = response.getOutputStream()) {
                in.transferTo(out);
                out.flush();
            }
            log.info("File downloaded successfully: {}", fileName);
        } catch (S3Exception | IOException e) {
            log.error(ErrorConstants.S3_UPLOAD_FAILED, fileName, e.getMessage());
            Object[] messageArgs = {fileName};
            throw new AdminPortalException(ErrorConstants.NOT_FOUND_ERROR_CODE, MessageFormat.format(ErrorConstants.NOT_FOUND_ERROR_MESSAGE, messageArgs));
        }
    }

    /**
     * Lists all files stored in the S3 bucket.
     *
     * @return a list of file keys present in the bucket
     */
    @Override
    public List<String> listObjects() {
        List<String> fileList = new ArrayList<>();
        try {
            log.info("Fetching list of files from S3 bucket...");
            ListObjectsV2Request listObjectsV2Request = ListObjectsV2Request.builder().bucket(s3Config.getBucket()).build();
            log.info("s3Client : {}", s3Client);
            log.info("listObjectsV2Request :{}", listObjectsV2Request);
            ListObjectsV2Response listObjectsV2Response = s3Client.listObjectsV2(listObjectsV2Request);

            listObjectsV2Response.contents().forEach(s3Object -> {
                log.info("Found file: {}", s3Object.key());
                fileList.add(s3Object.toString());
            });
        } catch (S3Exception e) {
            log.error("Failed to list objects in S3 bucket: {}", e.getMessage());
            throw new AdminPortalException(ErrorConstants.GENERIC_ERROR_CODE, e.getMessage());
        }
        return fileList;
    }


    /**
     * Reads file content from S3 and returns it as a ResponseBytes object.
     *
     * @param key the S3 file key
     * @return file content as ResponseBytes
     */
    @Override
    public InputStream readFile(String key) {
        try {
            log.info("Reading file from S3: {}", key);
            GetObjectRequest getObjectRequest = GetObjectRequest.builder().bucket(s3Config.getBucket()).key(key).build();
            return s3Client.getObjectAsBytes(getObjectRequest).asInputStream();

        } catch (S3Exception e) {
            log.error(ErrorConstants.S3_UPLOAD_FAILED, key, e.getMessage());
            throw new AdminPortalException(ErrorConstants.GENERIC_ERROR_CODE, e.getMessage());
        }
    }

    /**
     * Extract extension from Sandbox file content type and returns it as a string.
     *
     * @param mimeType sandbox file content type
     * @return file extension
     */
    public static String getExtensionFromMime(String mimeType) {
        if(ObjectUtils.isEmpty(mimeType) || !mimeType.contains("/")) {
            return null;
        }
        return mimeType.substring(mimeType.lastIndexOf("/") + 1).toLowerCase();
    }
}


////////

package com.epay.admin.portal.controller;


import com.epay.admin.portal.dto.admin.ChargebackUploadResponse;
import com.epay.admin.portal.externalservice.S3Service;
import com.epay.admin.portal.service.admin.AggCBRFileUploadService;
import com.sbi.epay.logging.utility.LoggerFactoryUtility;
import com.sbi.epay.logging.utility.LoggerUtility;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.Locale;

@RestController
@RequestMapping("/chargeback")
public class AggCBRFileUploadController {


    private final AggCBRFileUploadService chargebackService;
    private final LoggerUtility logger = LoggerFactoryUtility.getLogger(this.getClass());


    public AggCBRFileUploadController(AggCBRFileUploadService chargebackService) {
        this.chargebackService = chargebackService;
    }

    /**
     * Upload Chargeback Excel / CSV file
     */
    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<ChargebackUploadResponse> uploadChargebackFile(@RequestParam(value = "cardIssuer", required = false) String cardIssuer, @RequestParam(value = "cbStage", required = false) String cbStage, @RequestPart(value = "file", required = false) MultipartFile file) throws IOException {
        final S3Service s3Service;
        if (cardIssuer == null || cardIssuer.trim().isEmpty()) {
            logger.info("cbStage is required.");
        }

        if (cbStage == null || cbStage.trim().isEmpty()) {
            logger.info("cbStage is required.");
        }

        if (file == null || file.isEmpty()) {
            return ResponseEntity.badRequest().build();
        }
        if (!isValidFileExtension(file.getOriginalFilename())) {
            logger.info("Only .txt files are allowed");

            ChargebackUploadResponse errorresponse = ChargebackUploadResponse.builder().fileName(file.getOriginalFilename()).status("FAILED").recordsCount(String.valueOf("0")).reason("Only .csv / .xls , / .xlsx files are allowed").build();

            return ResponseEntity.badRequest().body(errorresponse);
        }


        ChargebackUploadResponse response = chargebackService.processFile(cardIssuer.trim(), cbStage.trim(), file);

        return ResponseEntity.ok(response);
    }

    private boolean isValidFileExtension(String fileName) {

        String lowerCaseFileName = fileName.toLowerCase(Locale.ROOT);
        // return lowerCaseFileName.endsWith(".txt");
        return lowerCaseFileName.endsWith(".xlsx") || lowerCaseFileName.endsWith(".xls") || lowerCaseFileName.endsWith(".csv") || lowerCaseFileName.endsWith(".txt");
    }

}



/////mplementation('org.apache.poi:poi-ooxml:5.4.1')
//////////////////
package com.epay.admin.portal.dto.admin;

import com.epay.admin.portal.entity.AuditEntityByDate;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Table;
import lombok.*;

import javax.persistence.Entity;
import javax.persistence.Id;
import java.util.UUID;


@EqualsAndHashCode(callSuper = true)
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Table
@Entity
public class ChargebackUploadResponse extends AuditEntityByDate {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    private String fileName;
    private String uploadUsername;
    private String path;
    private String remarks;
    private String status;
    private String recordsCount;
    private String reason;
}

///////////////////////////
package com.epay.admin.portal.service.admin;


import com.epay.admin.portal.dto.admin.ChargebackUploadResponse;
import com.epay.admin.portal.externalservice.S3Service;
import lombok.RequiredArgsConstructor;
import org.apache.poi.ss.usermodel.*;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Locale;

@Service
@RequiredArgsConstructor
public class AggCBRFileUploadService {
    private final S3Service s3Service;
    private final Path uploadDirectory = Paths.get("D:/chargeback/uploads");

    public ChargebackUploadResponse processFile(String cardIssuer, String cbStage, MultipartFile file) throws IOException {

        // Upload file to S3 using the unique upload number
        String s3Key = s3Service.uploadMultipartFile(file);
        String statusupd = "";
        String rsn = "";
      //  String savedFilePath = saveFile(file);
        long count = countRecords(file);
        if (count > 0) {
            statusupd = "UPLOADED";
            rsn = "File Upload Successfully";
        } else {
            statusupd = "FAILED";
            rsn = "file format incorrect";
        }
        return ChargebackUploadResponse.builder().fileName(file.getOriginalFilename()).uploadUsername("Username").path(s3Key).status(statusupd).remarks("remarks").reason(rsn).recordsCount(String.valueOf(count)).reason(rsn).build();
    }

    private long countExcelRecords(MultipartFile file) throws IOException {
        if (file == null || file.isEmpty()) {
            return 0;
        }

        String fileName = file.getOriginalFilename();

        if (fileName == null || fileName.isBlank()) {
            throw new IOException("Uploaded file name is empty");
        }

        System.out.println("File Name     : " + fileName);
        System.out.println("File Size     : " + file.getSize());
        System.out.println("Content Type  : " + file.getContentType());

        long count = 0;

        try (InputStream inputStream = file.getInputStream();
             Workbook workbook = WorkbookFactory.create(inputStream)) {

            System.out.println("Workbook created successfully");

            if (workbook.getNumberOfSheets() == 0) {
                throw new IOException("Excel file does not contain any sheet");
            }

            Sheet sheet = workbook.getSheetAt(0);

            System.out.println("Sheet Name    : " + sheet.getSheetName());
            System.out.println("Last Row Num  : " + sheet.getLastRowNum());

            // Row 0 = Header
            for (int i = 1; i <= sheet.getLastRowNum(); i++) {

                Row row = sheet.getRow(i);

                if (row == null) {
                    continue;
                }

                // Column A = Sr No
                Cell srNoCell = row.getCell(0);

                if (srNoCell == null) {
                    continue;
                }

                // If Sr No cell is blank, skip
                String srNo = new DataFormatter().formatCellValue(srNoCell);

                if (srNo == null || srNo.trim().isEmpty()) {
                    continue;
                }

                count++;
            }

        } catch (Exception e) {

            System.err.println("Excel reading failed");
            System.err.println("File Name : " + fileName);
            e.printStackTrace();

            throw e;
        }

        System.out.println("Excel Record Count : " + count);

        return count;
    }

    private long countCsvRecords(MultipartFile file) throws IOException {
        if (file == null || file.isEmpty()) {
            return 0;
        }
        byte[] fileBytes = file.getBytes();
        try (InputStream inputStream = new ByteArrayInputStream(fileBytes);

             BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, StandardCharsets.UTF_8))) {

            return reader.lines().skip(1).filter(line -> !line.trim().isEmpty()).count();

        }

    }

    private long countRecords(MultipartFile file) throws IOException {

        String fileName = file.getOriginalFilename();

        if (fileName == null) {
            return 0;
        }
        fileName = fileName.toLowerCase(Locale.ROOT);

        if (fileName.endsWith(".csv") || fileName.endsWith(".txt")) {
            return countCsvRecords(file);
        }

        if (fileName.endsWith(".xls") || fileName.endsWith(".xlsx")) {
            return countExcelRecords(file);
        }
        return 0;
    }

    private String saveFile(MultipartFile file) throws IOException {
        // create folder
        Files.createDirectories(uploadDirectory);
        String originalFileName = file.getOriginalFilename();

        if (originalFileName == null || originalFileName.isBlank()) {
            throw new IOException("File name is missing");
        }
        //Remove unsafe path
        String filename = Paths.get(originalFileName).getFileName().toString();
        Path targetFile = uploadDirectory.resolve(filename);

        //Save file

      // file.transferTo(targetFile.toFile());
        return targetFile.toString();
    }
}
//////////
 @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<ChargebackUploadResponse> uploadChargebackFile(@RequestParam(value = "cardIssuer", required = false) String cardIssuer, @RequestParam(value = "cbStage", required = false) String cbStage, @RequestPart(value = "file", required = false) MultipartFile file) throws IOException {
        final S3Service s3Service;
        if (cardIssuer == null || cardIssuer.trim().isEmpty()) {
            logger.info("cbStage is required.");
        }

        if (cbStage == null || cbStage.trim().isEmpty()) {
            logger.info("cbStage is required.");
        }

        if (file == null || file.isEmpty()) {
            return ResponseEntity.badRequest().build();
        }
        if (!isValidFileExtension(file.getOriginalFilename())) {
            logger.info("Only .txt files are allowed");

            ChargebackUploadResponse errorresponse = ChargebackUploadResponse.builder().fileName(file.getOriginalFilename()).status("FAILED").recordsCount(String.valueOf("0")).reason("Only .csv / .xls , / .xlsx files are allowed").build();

            return ResponseEntity.badRequest().body(errorresponse);
        }


        ChargebackUploadResponse response = chargebackService.processFile(cardIssuer.trim(), cbStage.trim(), file);

        return ResponseEntity.ok(response);
    }

    private boolean isValidFileExtension(String fileName) {

        String lowerCaseFileName = fileName.toLowerCase(Locale.ROOT);
        // return lowerCaseFileName.endsWith(".txt");
        return lowerCaseFileName.endsWith(".xlsx") || lowerCaseFileName.endsWith(".xls") || lowerCaseFileName.endsWith(".csv") || lowerCaseFileName.endsWith(".txt");
    }



////////////
private long countExcelRecords(MultipartFile file) throws IOException {

    if (file == null || file.isEmpty()) {
        return 0;
    }

    String fileName = file.getOriginalFilename();

    if (fileName == null || fileName.isBlank()) {
        throw new IOException("Uploaded file name is empty");
    }

    System.out.println("File Name     : " + fileName);
    System.out.println("File Size     : " + file.getSize());
    System.out.println("Content Type  : " + file.getContentType());

    long count = 0;

    try (InputStream inputStream = file.getInputStream();
         Workbook workbook = WorkbookFactory.create(inputStream)) {

        System.out.println("Workbook created successfully");

        if (workbook.getNumberOfSheets() == 0) {
            throw new IOException("Excel file does not contain any sheet");
        }

        Sheet sheet = workbook.getSheetAt(0);

        System.out.println("Sheet Name    : " + sheet.getSheetName());
        System.out.println("Last Row Num  : " + sheet.getLastRowNum());

        // Row 0 = Header
        for (int i = 1; i <= sheet.getLastRowNum(); i++) {

            Row row = sheet.getRow(i);

            if (row == null) {
                continue;
            }

            // Column A = Sr No
            Cell srNoCell = row.getCell(0);

            if (srNoCell == null) {
                continue;
            }

            // If Sr No cell is blank, skip
            String srNo = new DataFormatter().formatCellValue(srNoCell);

            if (srNo == null || srNo.trim().isEmpty()) {
                continue;
            }

            count++;
        }

    } catch (Exception e) {

        System.err.println("Excel reading failed");
        System.err.println("File Name : " + fileName);
        e.printStackTrace();

        throw e;
    }

    System.out.println("Excel Record Count : " + count);

    return count;
}

****
package com.epay.admin.portal.service.admin;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.List;
import java.util.Locale;
import java.util.stream.Stream;

import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import com.epay.admin.portal.dto.admin.ChargebackUploadResponse;
import com.epay.admin.portal.dto.admin.ExcelParser;

import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class AggCBRFFileUploadService {

    private final S3Service s3Service;

    private final ExcelParser excelParser;

    /*
     * Local folder where uploaded files will be saved
     */
    private final Path uploadDirectory =
            Paths.get("D:/chargeback/uploads");


    /**
     * Main file processing method
     */
    public ChargebackUploadResponse processFile(
            String cardIssuer,
            String cbStage,
            MultipartFile file) throws IOException {

        /*
         * Validate file
         */
        if (file == null || file.isEmpty()) {

            return ChargebackUploadResponse.builder()
                    .fileName(file != null
                            ? file.getOriginalFilename()
                            : null)
                    .uploadUsername("Username")
                    .status("FAILED")
                    .reason("File is empty")
                    .build();
        }

        String fileName = file.getOriginalFilename();

        if (fileName == null || fileName.isBlank()) {

            return ChargebackUploadResponse.builder()
                    .fileName(null)
                    .uploadUsername("Username")
                    .status("FAILED")
                    .reason("File name is missing")
                    .build();
        }


        /*
         * Validate extension
         */
        String lowerFileName =
                fileName.toLowerCase(Locale.ROOT);

        boolean validFile =
                lowerFileName.endsWith(".xls")
                || lowerFileName.endsWith(".xlsx")
                || lowerFileName.endsWith(".csv")
                || lowerFileName.endsWith(".txt");

        if (!validFile) {

            return ChargebackUploadResponse.builder()
                    .fileName(fileName)
                    .uploadUsername("Username")
                    .status("FAILED")
                    .reason("Invalid file format")
                    .build();
        }


        /*
         * Save file locally
         */
        String savedFilePath = saveFile(file);


        /*
         * Count valid records
         */
        long count = countRecords(file);


        /*
         * Upload to S3
         *
         * Keep this according to your existing S3Service
         */
        String s3Key =
                s3Service.uploadMultipartFile(file);


        /*
         * Prepare response
         */
        String statusupd;
        String rsn;

        if (count > 0) {

            statusupd = "UPLOADED";
            rsn = "File Upload Successfully";

        } else {

            statusupd = "FAILED";
            rsn = "File format incorrect";
        }


        /*
         * Return response DTO
         */
        return ChargebackUploadResponse.builder()
                .fileName(fileName)
                .uploadUsername("Username")
                .path(s3Key)
                .status(statusupd)
                .reason(rsn)
                .build();
    }


    /**
     * Decide whether CSV/TXT or Excel
     */
    private long countRecords(MultipartFile file)
            throws IOException {

        if (file == null || file.isEmpty()) {
            return 0;
        }

        String fileName =
                file.getOriginalFilename();

        if (fileName == null) {
            return 0;
        }

        fileName =
                fileName.toLowerCase(Locale.ROOT);


        /*
         * CSV / TXT
         */
        if (fileName.endsWith(".csv")
                || fileName.endsWith(".txt")) {

            return countCsvRecords(file);
        }


        /*
         * Excel
         */
        if (fileName.endsWith(".xls")
                || fileName.endsWith(".xlsx")) {

            return countExcelRecords(file);
        }


        return 0;
    }


    /**
     * Count Excel records using ExcelParser
     *
     * ExcelParser already skips row 0 (header)
     */
    private long countExcelRecords(MultipartFile file)
            throws IOException {

        if (file == null || file.isEmpty()) {
            return 0;
        }

        String fileName =
                file.getOriginalFilename();

        /*
         * Use your ExcelParser here
         */
        List<String[]> rows =
                excelParser.parseFile(
                        file.getInputStream(),
                        fileName
                );

        return rows.size();
    }


    /**
     * Count CSV/TXT records
     *
     * First line = Header
     */
    private long countCsvRecords(MultipartFile file)
            throws IOException {

        if (file == null || file.isEmpty()) {
            return 0;
        }

        try (
                InputStream inputStream =
                        file.getInputStream();

                BufferedReader reader =
                        new BufferedReader(
                                new InputStreamReader(
                                        inputStream,
                                        StandardCharsets.UTF_8
                                )
                        )
        ) {

            /*
             * Read header
             */
            reader.readLine();

            /*
             * Count non-empty records
             */
            long count = 0;

            String line;

            while ((line = reader.readLine()) != null) {

                if (!line.trim().isEmpty()) {
                    count++;
                }
            }

            return count;
        }
    }


    /**
     * Save uploaded file to local folder
     */
    private String saveFile(MultipartFile file)
            throws IOException {

        /*
         * Create folder if it doesn't exist
         */
        Files.createDirectories(uploadDirectory);


        /*
         * Get original file name
         */
        String originalFileName =
                file.getOriginalFilename();

        if (originalFileName == null
                || originalFileName.isBlank()) {

            throw new IOException(
                    "File name is missing"
            );
        }


        /*
         * Remove unsafe path
         *
         * Example:
         * ../../test.xlsx
         *
         * becomes:
         * test.xlsx
         */
        String fileName =
                Paths.get(originalFileName)
                        .getFileName()
                        .toString();


        /*
         * Create target path
         */
        Path targetFile =
                uploadDirectory.resolve(fileName);


        /*
         * Save file
         */
        file.transferTo(targetFile.toFile());


        return targetFile.toString();
    }
}


*******






Document Title:
NON-COLLECTION OF GST AMOUNT FROM CUSTOMER FOR INTL CARD TRANSACTIONS

Document Number:
GIITC/ePay&PG/SBIePay/NON-COLLECTION OF GST AMOUNT FROM CUSTOMER FOR INTL CARD TRANSACTIONS

Document Status:

Version Number:
1.0

Release Date:

------------------------------------------------------------
Revision Details
------------------------------------------------------------

Version No. : 1.0
Date        : 08-04-2026
Particulars : Initial Document
Approved By :

------------------------------------------------------------
Document Contact Details
------------------------------------------------------------

Role        : Author
Name        : Bhagyashri Tikone
Designation : Developer

Role        : Author
Name        : Shital Patil
Designation : Technical Lead

Role        : Reviewer
Name        : Santosh Kumar Sahoo
Designation : Solution Architect

Role        : Approver
Name        : Anand Sharma
Designation : Project Manager

Role        : Approver
Name        :
Designation : BU Team Head

Role        : Custodian

Distribution List:.





MerchantPricingRequest request = new MerchantPricingRequest();
request.setMId("TEST_MID");
request.setTransactionAmount(new BigDecimal("100.00"));
request.setPayModeCode("UPI");
request.setGtwMapsId(1L);
request.setPayProcType("SALE");
request.setAtrn("123456");
request.setPostAmount(new BigDecimal("100.00"));



@ExtendWith(MockitoExtension.class)
class AdminDaoTest {

    @InjectMocks
    private AdminDao adminDao;

    @Mock
    private AdminServicesClient adminServicesClient;

    @Mock
    private ErrorLogDao errorLogDao;

    @Mock
    private PricingMapper pricingMapper;
}

@Test
void getMerchantByMId_success() {

    String mId = "100001";

    MerchantInfoResponse merchantInfo = new MerchantInfoResponse();

    TransactionResponse<MerchantInfoResponse> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);
    response.setData(List.of(merchantInfo));

    when(adminServicesClient.getMerchantInfoByMid(mId))
            .thenReturn(response);

    MerchantInfoResponse result =
            adminDao.getMerchantByMId(mId);

    assertNotNull(result);
    assertEquals(merchantInfo, result);
}
@Test
void getMerchantByMId_shouldThrowTransactionException_whenErrorsExist() {

    String mId = "100001";

    ErrorResponse error = new ErrorResponse();

    TransactionResponse<MerchantInfoResponse> response =
            new TransactionResponse<>();

    response.setErrors(List.of(error));

    when(adminServicesClient.getMerchantInfoByMid(mId))
            .thenReturn(response);

    assertThrows(
            TransactionException.class,
            () -> adminDao.getMerchantByMId(mId));
}

@Test
void getMerchantRFCDetails_success() {

    String mId = "100001";

    MerchantRfcResponse rfcResponse =
            new MerchantRfcResponse();

    TransactionResponse<MerchantRfcResponse> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);
    response.setData(List.of(rfcResponse));

    when(adminServicesClient.getMerchantRFCInfo(mId))
            .thenReturn(response);

    MerchantRfcResponse result =
            adminDao.getMerchantRFCDetails(mId);

    assertEquals(rfcResponse, result);
}

@Test
void getMerchantRFCDetails_shouldThrowTransactionException() {

    TransactionResponse<MerchantRfcResponse> response =
            new TransactionResponse<>();

    response.setErrors(List.of(new ErrorResponse()));

    when(adminServicesClient.getMerchantRFCInfo(anyString()))
            .thenReturn(response);

    assertThrows(TransactionException.class,
            () -> adminDao.getMerchantRFCDetails("100001"));
}

@Test
void getMerchantMultiAccount_success() {

    TransactionResponse<String> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);
    response.setData(List.of("ACC1", "ACC2"));

    when(adminServicesClient.getMultiAccountDetailsApi("100001"))
            .thenReturn(response);

    List<String> result =
            adminDao.getMerchantMultiAccount("100001");

    assertEquals(2, result.size());
}
@Test
void getMerchantMultiAccount_notFound() {

    TransactionResponse<String> response =
            new TransactionResponse<>();

    when(adminServicesClient.getMultiAccountDetailsApi(anyString()))
            .thenReturn(response);

    assertThrows(TransactionException.class,
            () -> adminDao.getMerchantMultiAccount("100001"));
}
@Test
void getMerchantPayModes_success() {

    ObjectMapper mapper = new ObjectMapper();

    ObjectNode root = mapper.createObjectNode();
    ArrayNode payModes = mapper.createArrayNode();

    payModes.add("CC");
    payModes.add("DC");

    root.set("payModes", payModes);

    TransactionResponse<JsonNode> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);
    response.setData(List.of(root));

    when(adminServicesClient.getMerchantPayModeInfo("100001"))
            .thenReturn(response);

    JsonNode result =
            adminDao.getMerchantPayModes("100001");

    assertEquals(2, result.size());
}
@Test
void getMerchantVVLDetails_success() {

    MerchantVvlResponse vvl = new MerchantVvlResponse();

    TransactionResponse<MerchantVvlResponse> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);
    response.setData(List.of(vvl));

    when(adminServicesClient.getVvlDetails("100001"))
            .thenReturn(response);

    List<MerchantVvlResponse> result =
            adminDao.getMerchantVVLDetails("100001");

    assertEquals(1, result.size());
}
@Test
void getGatewayConfigDetails_success() {

    GatewayConfigDetailsResponse config =
            new GatewayConfigDetailsResponse();

    TransactionResponse<GatewayConfigDetailsResponse> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);
    response.setData(List.of(config));

    when(adminServicesClient.getGatewayConfigDetails(
            anyString(),
            anyString(),
            anyString()))
            .thenReturn(response);

    GatewayConfigDetailsResponse result =
            adminDao.getGatewayConfigDetails(
                    "100001",
                    "228",
                    "DC");

    assertEquals(config, result);
}

@Test
void isValidCurrencyCode_success() {

    TransactionResponse<?> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);

    when(adminServicesClient.getCurrencyValidate(any()))
            .thenReturn(response);

    assertTrue(
            adminDao.isValidCurrencyCode("100001", "INR"));
}
@Test
void isValidCurrencyCode_failure() {

    TransactionResponse<?> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_FAILURE);

    when(adminServicesClient.getCurrencyValidate(any()))
            .thenReturn(response);

    assertFalse(
            adminDao.isValidCurrencyCode("100001", "INR"));
}
@Test
void isValidChannelBank_success() {

    TransactionResponse<?> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_SUCCESS);

    when(adminServicesClient.getChannelBankValidate(
            anyString(),
            anyString()))
            .thenReturn(response);

    assertTrue(
            adminDao.isValidChannelBank("228", "HDFC"));
}

@Test
void isValidChannelBank_failure() {

    TransactionResponse<?> response =
            new TransactionResponse<>();

    response.setStatus(TransactionConstant.RESPONSE_FAILURE);

    when(adminServicesClient.getChannelBankValidate(
            anyString(),
            anyString()))
            .thenReturn(response);

    assertFalse(
            adminDao.isValidChannelBank("228", "HDFC"));
}

@Test
void getValidMerchantPricingStructure_success() {

    MerchantPricingRequest request =
            new MerchantPricingRequest();

    MerchantPricingResponse pricingResponse =
            new MerchantPricingResponse();

    pricingResponse.setFeeProcessingFlag("Y");
    pricingResponse.setBearableComponent("F");

    MerchantPricingResponse result =
            adminDao.getValidMerchantPricingStructure(request);

    assertNotNull(result);
}
@Test
void getValidMerchantPricingStructure_invalidFeeProcessingFlag() {

    MerchantPricingRequest request =
            new MerchantPricingRequest();

    ValidationException ex =
            assertThrows(
                    ValidationException.class,
                    () -> adminDao.getValidMerchantPricingStructure(request));

    assertNotNull(ex);
}
@Test
void getValidMerchantPricingStructure_invalidBearableComponent() {

    MerchantPricingRequest request =
            new MerchantPricingRequest();

    assertThrows(
            ValidationException.class,
            () -> adminDao.getValidMerchantPricingStructure(request));
}
@Test
void getValidMerchantPricingStructure_responseErrors() {

    MerchantPricingRequest request =
            new MerchantPricingRequest();

    assertThrows(
            TransactionException.class,
            () -> adminDao.getValidMerchantPricingStructure(request));
}




======
public static final String PAY_PROC_TYPE_DOM = "DOM";

public static final String PAY_MODE_CC = "CC";
public static final String PAY_MODE_DC = "DC";
public static final String PAY_MODE_PC = "PC";
public static final String PAY_MODE_SCC = "SCC";

public static final BigDecimal GST_EXEMPT_AMOUNT =
        BigDecimal.valueOf(2000);




RU=https://epay.sbi.bank.in/payagg/...

https://billing.mahadiscom.in/sbiSuccessHandler.php

Request Body:
{
  "bankId": "013",
  "pid": "000002227446",
  "itc": "MSEDCL LT Post Paid Bill M-App",
  "prn": "5292093528637",
  "amt": "18110.00",
  "crn": "INR"
} 

# Checksum generated

# Customer Redirect to-
https://epay.sbi.bank.in/payagg/...

# customer Select -
Bank Of India Net Banking

# SBIePay sends response-

{ PAID=Y
 AMT=18110
 PRN=5292093528637} 
 
 # Double Verification response 

 # Checksum Validation

 # Update Transaction Table

 # Response Received



Customer
   |
   |
Merchant Application
   |
Generate ATRN
Generate Checksum
   |
   V
SBIePay Gateway
   |
Select BOI Net Banking
   |
Customer Pays
   |
   V
SBIePay Response
   |
Validate Checksum
   |
Double Verification API
   |
Update DB
   |
Success Callback URL
   |
Merchant Response








========================>
String response = """
{
  "status": 1,
  "data": [
    {
      "mId": 1000003,
      "payModeCode": "DC",
      "gtwMapsId": 228,
      "payProcType": "ONUS",
      "merchantFee": 0,
      "instructionType": "I",
      "slabFrom": 0,
      "slabTo": 999999999999,
      "merchantFeeApplicable": "Y",
      "merchantFeeType": "F",
      "otherFeeApplicable": "N",
      "otherFeeType": "F",
      "otherFee": 0,
      "gtwFeeApplicable": "N",
      "gtwFeeType": "F",
      "gtwFee": 0,
      "aggServiceFeeApplicable": "N",
      "aggServiceFeeType": "F",
      "aggServiceFee": 0,
      "feeProcessingFlag": "H",
      "serviceTax": 18,
      "serviceTaxType": "P",
      "serviceTaxId": 1,
      "txnApplicable": "Y",
      "transactionType": "ORDER",
      "bearableComponent": "FEE",
      "bearableEntity": "C",
      "bearableAmountCutoff": 0,
      "bearableFlatRate": 0.1,
      "bearableLimit": "NA",
      "bearablePercentageRate": 0,
      "bearableType": "F",
      "totalFeeRate": 0.1,
      "processFlag": "N"
    }
  ],
  "count": null,
  "total": null,
  "errors": null
}
""";





https://teams.microsoft.com/l/chat/19:meeting_YTA2NzEwNmEtZWY5Ni00Mzc4LWFhNzAtMDRhNGJmYzZiZDBi@thread.v2/conversations?context=%7B%22contextType%22%3A%22chat%22%7D










private BigDecimal calculateGstAmount(
        MerchantPricingResponse pricingStructure,
        MerchantPricingRequest request,
        MerchantPricingDto pricingDto) {

    // DOM transaction <= 2000 => No GST
    if ("DOM".equalsIgnoreCase(pricingStructure.getPayProcType())
            && request.getPostAmount()
                      .compareTo(BigDecimal.valueOf(2000)) <= 0) {

        return BigDecimal.ZERO;
    }

    BigDecimal aggFee = pricingDto.getAggServiceFeeAb();

    if (aggFee == null) {
        return BigDecimal.ZERO;
    }

    // GST = 18%
    return aggFee.multiply(BigDecimal.valueOf(18))
                 .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);
}







private BigDecimal calculateGstAmount(
        MerchantPricingResponse pricingStructure,
        MerchantPricingDto pricingDto) {

    if (pricingStructure.getServiceTax() == null) {
        return BigDecimal.ZERO;
    }

    if (pricingDto.getAggServiceFeeAb() == null) {
        return BigDecimal.ZERO;
    }

    // Percentage GST
    if ("P".equalsIgnoreCase(pricingStructure.getServiceTaxType())) {

        return pricingDto.getAggServiceFeeAb()
                .multiply(pricingStructure.getServiceTax())
                .divide(BigDecimal.valueOf(100),
                        2,
                        RoundingMode.HALF_UP);
    }

    // Flat GST
    if ("F".equalsIgnoreCase(pricingStructure.getServiceTaxType())) {
        return pricingStructure.getServiceTax();
    }

    return BigDecimal.ZERO;
}






private BigDecimal calculateGstAmount(
        MerchantPricingResponse pricingStructure,
        MerchantPricingDto dto) {

    if (pricingStructure.getServiceTax() == null) {
        return BigDecimal.ZERO;
    }

    if (dto.getAggServiceFeeAb() == null) {
        return BigDecimal.ZERO;
    }

    if ("P".equalsIgnoreCase(
            pricingStructure.getServiceTaxType())) {

        return dto.getAggServiceFeeAb()
                .multiply(pricingStructure.getServiceTax())
                .divide(BigDecimal.valueOf(100),
                        2,
                        RoundingMode.HALF_UP);
    }

    return pricingStructure.getServiceTax();
}




MerchantPricingDto merchantPricingDto =
        calculateFee(pricingStructure,
                     merchantPricingRequest);

BigDecimal gstAmount =
        calculateGST(
                pricingStructure,
                merchantPricingDto);

merchantPricingDto.setGstAmount(gstAmount);

checkAtrnAndSavePricingInfo(
        merchantPricingRequest,
        merchantPricingDto,
        pricingStructure);

return buildMerchantFeeResponse(
        merchantPricingRequest,
        merchantPricingDto);





private BigDecimal calculateGST(
        MerchantPricingResponse pricingStructure,
        MerchantPricingDto pricingDto) {

    if (pricingStructure.getServiceTax() == null) {
        return BigDecimal.ZERO;
    }

    if (pricingDto.getAggServiceFeeAb() == null) {
        return BigDecimal.ZERO;
    }

    return pricingDto.getAggServiceFeeAb()
            .multiply(pricingStructure.getServiceTax())
            .divide(BigDecimal.valueOf(100),
                    2,
                    RoundingMode.HALF_UP);
}









import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.text.MessageFormat;
import java.util.Optional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderDaoTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderDao orderDao;

    private TransactionDto transactionDto;
    private Order order;

    @BeforeEach
    void setUp() {

        transactionDto = new TransactionDto();
        transactionDto.setSbiOrderRefNumber("SBI123");
        transactionDto.setOrderRefNumber("ORD123");

        order = new Order();
    }

    @Test
    void testGetOrderDetails_Success() {

        when(orderRepository
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber()))
                .thenReturn(Optional.of(order));

        Order result = orderDao.getOrderDetails(transactionDto);

        assertNotNull(result);
        assertEquals(order, result);

        verify(orderRepository, times(1))
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber());
    }

    @Test
    void testGetOrderDetails_OrderNotFound() {

        when(orderRepository
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber()))
                .thenReturn(Optional.empty());

        PaymentException exception = assertThrows(
                PaymentException.class,
                () -> orderDao.getOrderDetails(transactionDto));

        assertEquals(
                ErrorConstants.NOT_FOUND_ERROR_CODE,
                exception.getErrorCode());

        assertEquals(
                MessageFormat.format(
                        ErrorConstants.NOT_FOUND_ERROR_MESSAGE,
                        "Order"),
                exception.getMessage());

        verify(orderRepository, times(1))
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber());
    }
}








import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.nio.charset.StandardCharsets;
import java.sql.Timestamp;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OfflinePaymentDaoTest {

    @Mock
    private OfflinePaymentRepository offlinePaymentRepository;

    @Mock
    private CashManagementMapper mapper;

    @InjectMocks
    private OfflinePaymentDao offlinePaymentDao;

    @Mock
    private TransactionDto transactionDto;

    @Mock
    private PlainCashChallanJsonRequest plainCashChallanJsonRequest;

    @Mock
    private CashChallanPaymentFinalResponseDto cashChallanPaymentFinalResponseDto;

    @Mock
    private NeftPaymentFinalResponseDto neftPaymentFinalResponseDto;

    @Mock
    private CashManagementDto cashManagementDto;

    private UUID reportManagementId;

    @BeforeEach
    void setUp() {

        reportManagementId = UUID.randomUUID();

        when(transactionDto.getMerchantId()).thenReturn(1001L);
        when(transactionDto.getAtrnNum()).thenReturn("ATRN12345");
        when(transactionDto.getPayMode()).thenReturn("CASH");

        when(plainCashChallanJsonRequest.getNameOfTheCustomer())
                .thenReturn("Vaibhav");

        when(plainCashChallanJsonRequest.getMobileNumber())
                .thenReturn("9876543210");

        when(plainCashChallanJsonRequest.getEmailId())
                .thenReturn("test@gmail.com");

        when(plainCashChallanJsonRequest.getChallanGenerationDateAndTime())
                .thenReturn(new Timestamp(System.currentTimeMillis()));

        when(plainCashChallanJsonRequest.getChallanExpiryOn())
                .thenReturn(new Timestamp(System.currentTimeMillis()));

        when(cashManagementDto.getAtrnNum())
                .thenReturn("ATRN12345");
    }

    @Test
    void testSaveCashPdfDetails() {

        assertDoesNotThrow(() ->
                offlinePaymentDao.saveCashPdfDetails(
                        reportManagementId,
                        ReportStatus.SUCCESS,
                        transactionDto,
                        cashChallanPaymentFinalResponseDto,
                        plainCashChallanJsonRequest
                ));

        verify(offlinePaymentRepository, times(1))
                .save(any(OfflinePaymentModeDetailsEntity.class));

        verify(mapper, times(1))
                .mapEntityToDto(any(OfflinePaymentModeDetailsEntity.class));
    }

    @Test
    void testFindByAtrnNum() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        CashManagementDto dto = new CashManagementDto();

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        when(mapper.mapEntityToDto(entity))
                .thenReturn(dto);

        offlinePaymentDao.findByAtrnNum("ATRN12345");

        verify(offlinePaymentRepository, times(1))
                .findByAtrnNum("ATRN12345");

        verify(mapper, times(1))
                .mapEntityToDto(entity);
    }

    @Test
    void testUpdateCashPdfDetails() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        assertDoesNotThrow(() ->
                offlinePaymentDao.updateCashPdfDetails(
                        "test.pdf",
                        cashManagementDto
                ));

        verify(offlinePaymentRepository, times(1))
                .save(entity);
    }

    @Test
    void testUpdateCashPdfBlobDetails() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        assertDoesNotThrow(() ->
                offlinePaymentDao.updateCashPdfBlobDetails(
                        "BASE64DATA",
                        cashManagementDto,
                        "test.pdf"
                ));

        verify(offlinePaymentRepository, times(1))
                .save(entity);
    }

    @Test
    void testUpdateCashPdfBlobData() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        byte[] fileData = "PDF_DATA".getBytes(StandardCharsets.UTF_8);

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        assertDoesNotThrow(() ->
                offlinePaymentDao.updateCashPdfBlobData(
                        fileData,
                        cashManagementDto,
                        "test.pdf"
                ));

        verify(offlinePaymentRepository, times(1))
                .save(entity);
    }

    @Test
    void testSaveNeftPdfDetails() {

        assertDoesNotThrow(() ->
                offlinePaymentDao.saveNeftPdfDetails(
                        reportManagementId,
                        ReportStatus.SUCCESS,
                        transactionDto,
                        neftPaymentFinalResponseDto,
                        plainCashChallanJsonRequest
                ));

        verify(offlinePaymentRepository, times(1))
                .save(any(OfflinePaymentModeDetailsEntity.class));

        verify(mapper, times(1))
                .mapEntityToDto(any(OfflinePaymentModeDetailsEntity.class));
    }
}









import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.lang.reflect.Method;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class NotificationDaoTest {

    @InjectMocks
    private NotificationDao notificationDao;

    @Mock
    private NotificationMapper notificationMapper;

    @Mock
    private NotificationManagementRepository notificationManagementRepository;

    @Mock
    private SmsNotificationProducer smsNotificationProducer;

    @Mock
    private EmailNotificationProducer emailNotificationProducer;

    @Mock
    private EmailService emailService;

    @Mock
    private CashChallanConfigDetails cashChallanConfigDetails;

    @Mock
    private SmsService smsService;

    @BeforeEach
    void setUp() {
    }

    @Test
    void testPublishSmsNotification() {

        CashSmsDto cashSmsDto = new CashSmsDto();
        cashSmsDto.setRequestType("SMS");

        String routingKey = "sms-routing";

        notificationDao.publishSmsNotification(cashSmsDto, routingKey);

        verify(smsNotificationProducer)
                .publish(cashSmsDto.getRequestType(), routingKey, cashSmsDto);
    }

    @Test
    void testSendSmsNotification() {

        CashSmsDto cashSmsDto = new CashSmsDto();

        SmsDto smsDto = new SmsDto();

        when(notificationMapper.mapSmsDtoToToDto(any(CashSmsDto.class)))
                .thenReturn(smsDto);

        assertDoesNotThrow(() ->
                notificationDao.sendSmsNotification(cashSmsDto));

        verify(notificationMapper)
                .mapSmsDtoToToDto(cashSmsDto);

        verify(smsService)
                .sendSMS(smsDto);
    }

    @Test
    void testSendSMS_Exception() throws Exception {

        SmsDto smsDto = new SmsDto();

        doThrow(new RuntimeException("SMS Exception"))
                .when(smsService)
                .sendSMS(any(SmsDto.class));

        Method method = NotificationDao.class
                .getDeclaredMethod("sendSMS", SmsDto.class);

        method.setAccessible(true);

        method.invoke(notificationDao, smsDto);

        // verify service called
        verify(smsService).sendSMS(smsDto);
    }

    @Test
    void testBuildNotificationManagement_ForSms() throws Exception {

        CashSmsDto cashSmsDto = new CashSmsDto();

        cashSmsDto.setRequestType("SMS");

        UUID entityId = UUID.randomUUID();
        cashSmsDto.setEntityId(entityId);

        Method method = NotificationDao.class
                .getDeclaredMethod("buildNotificationManagement",
                        CashSmsDto.class);

        method.setAccessible(true);

        NotificationManagement result =
                (NotificationManagement) method.invoke(notificationDao,
                        cashSmsDto);

        assertEquals("SMS", result.getRequestType());
        assertEquals(entityId, result.getEntityId());
    }

    @Test
    void testPublishEmailNotification() {

        CashEmailDto cashEmailDto = new CashEmailDto();

        cashEmailDto.setRequestType("EMAIL");

        UUID entityId = UUID.randomUUID();

        String routingKey = "email-routing";

        notificationDao.publishEmailNotification(
                cashEmailDto,
                routingKey,
                entityId
        );

        verify(emailNotificationProducer)
                .publish(
                        cashEmailDto.getRequestType(),
                        routingKey,
                        cashEmailDto
                );
    }

    @Test
    void testGetEmailDto_WhenRecipientPresent() throws Exception {

        CashEmailDto cashEmailDto = new CashEmailDto();

        UUID userId = UUID.randomUUID();

        when(cashChallanConfigDetails.getFrom())
                .thenReturn("test@gmail.com");

        when(cashChallanConfigDetails.getRecipient())
                .thenReturn("receiver@gmail.com");

        Method method = NotificationDao.class
                .getDeclaredMethod(
                        "getEmailDto",
                        CashEmailDto.class,
                        UUID.class
                );

        method.setAccessible(true);

        method.invoke(notificationDao, cashEmailDto, userId);

        assertEquals("test@gmail.com", cashEmailDto.getFrom());
        assertEquals(userId, cashEmailDto.getEntityId());
        assertEquals("receiver@gmail.com", cashEmailDto.getRecipient());
    }

    @Test
    void testSendEmailNotification() {

        CashEmailDto cashEmailDto = new CashEmailDto();

        EmailDto emailDto = new EmailDto();

        when(notificationMapper.mapEmailDtoToDto(any(CashEmailDto.class)))
                .thenReturn(emailDto);

        assertDoesNotThrow(() ->
                notificationDao.sendEmailNotification(cashEmailDto));

        verify(notificationMapper)
                .mapEmailDtoToDto(cashEmailDto);

        verify(emailService)
                .sendEmail(emailDto);
    }

    @Test
    void testSendEmail_Exception() throws Exception {

        EmailDto emailDto = new EmailDto();

        doThrow(new RuntimeException("Email Exception"))
                .when(emailService)
                .sendEmail(any(EmailDto.class));

        Method method = NotificationDao.class
                .getDeclaredMethod("sendEmail", EmailDto.class);

        method.setAccessible(true);

        method.invoke(notificationDao, emailDto);

        verify(emailService).sendEmail(emailDto);
    }
}









@Test
void testProcessInbDoubleVerRequest_StringResponse() {

    NbMapWebResponseDto nbMapWebResponseDto = new NbMapWebResponseDto();
    NbMapDVResponseDto nbMapDVResponseDto = new NbMapDVResponseDto();

    NbMapWebResponseDto response =
            nbPaymentDao.processInbDoubleVerRequest(
                    nbMapWebResponseDto,
                    nbMapDVResponseDto,
                    "encryptedResponse");

    assertNotNull(response);
}







import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.math.BigDecimal;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class NbPaymentDaoTest {

    @Mock
    private PaymentDao paymentDao;

    @Mock
    private InbEncryptionDecryptionUtil inbEncryptionDecryptionUtil;

    @Mock
    private PaymentValidator paymentValidator;

    @Mock
    private StatusUpdatePaymentDao statusUpdatePaymentDao;

    @InjectMocks
    private NbPaymentDao nbPaymentDao;

    private NbMapWebResponseDto nbMapWebResponseDto;
    private NbMapWebResponseDto nbMapDVResponseDto;

    @BeforeEach
    void setUp() {

        nbMapWebResponseDto = new NbMapWebResponseDto();
        nbMapWebResponseDto.setTxnrefNo("TXN123");
        nbMapWebResponseDto.setSbirefNo("SBI123");
        nbMapWebResponseDto.setAmount(new BigDecimal("100"));

        nbMapDVResponseDto = new NbMapWebResponseDto();
        nbMapDVResponseDto.setTxnrefNo("TXN123");
        nbMapDVResponseDto.setSbirefNo("SBI123");
        nbMapDVResponseDto.setAmount(new BigDecimal("100"));
        nbMapDVResponseDto.setCheckSum("checksum");
    }

    @Test
    void testProcessInbDoubleVerRequest_Success() {

        nbMapDVResponseDto.setStatus(PaymentConstants.SUCCESS_CONST);

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentSuccessPendingStatusUpdate(
                        eq(PaymentStatus.SUCCESS),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testProcessInbDoubleVerRequest_Pending() {

        nbMapDVResponseDto.setStatus(PaymentConstants.PENDING_CONST);

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentSuccessPendingStatusUpdate(
                        eq(PaymentStatus.PENDING),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testProcessInbDoubleVerRequest_Failure() {

        nbMapDVResponseDto.setStatus(PaymentConstants.FAILURE_CONST);

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testProcessInbDoubleVerRequest_DefaultCase() {

        nbMapDVResponseDto.setStatus("UNKNOWN");

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testValidateCallBackResponse_ChecksumFail() {

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(false);

        assertThrows(
                PaymentException.class,
                () -> nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse"));
    }

    @Test
    void testValidateCallBackResponse_AmountMismatch() {

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(false);

        assertThrows(
                PaymentException.class,
                () -> nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse"));
    }

    @Test
    void testValidateCallBackResponse_BankRefMismatch() {

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(false);

        assertThrows(
                PaymentException.class,
                () -> nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse"));
    }

    @Test
    void testProcessInbDoubleVerRequest_StringResponse() {

        String response =
                nbPaymentDao.processInbDoubleVerRequest(nbMapWebResponseDto);

        assertNotNull(response);
    }
}




import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import com.epay.payment.constant.PaymentConstants;
import com.epay.payment.dao.MobikwikWalletPaymentDao;
import com.epay.payment.dao.PaymentDao;
import com.epay.payment.dao.StatusUpdatePaymentDao;
import com.epay.payment.dto.MobikwikWalletDvResponseDto;
import com.epay.payment.dto.MobikwikWalletMapWebResponseDto;
import com.epay.payment.util.MobikwikWalletEncryptionDecryptionUtil;
import com.epay.payment.util.PaymentUtil;
import com.epay.payment.validator.PaymentValidator;
import com.epay.payment.config.WalletConfig;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.Mockito;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class MobikwikWalletPaymentDaoTest {

    @Mock
    private PaymentDao paymentDao;

    @Mock
    private MobikwikWalletEncryptionDecryptionUtil walletEncryptionDecryptionUtil;

    @Mock
    private PaymentValidator paymentValidator;

    @Mock
    private StatusUpdatePaymentDao statusUpdatePaymentDao;

    @Mock
    private WalletConfig walletConfig;

    @Mock
    private PaymentUtil paymentUtil;

    @InjectMocks
    private MobikwikWalletPaymentDao mobikwikWalletPaymentDao;

    private MobikwikWalletMapWebResponseDto webResponseDto;
    private MobikwikWalletDvResponseDto dvResponseDto;

    @BeforeEach
    void setUp() {

        webResponseDto = new MobikwikWalletMapWebResponseDto();
        webResponseDto.setMid("MID123");
        webResponseDto.setOrderid("ORD123");

        dvResponseDto = new MobikwikWalletDvResponseDto();
        dvResponseDto.setOrderid("ORD123");
        dvResponseDto.setTxid("TXN123");
    }

    @Test
    void testProcessWalletDoubleVerRequest_StringResponse() {

        Mockito.spy(mobikwikWalletPaymentDao);

        String expected = "CALLBACK_RESPONSE";

        when(mobikwikWalletPaymentDao.getCallBackResponse("MID123", "ORD123"))
                .thenReturn(expected);

        String actual =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(webResponseDto);

        assertEquals(expected, actual);
    }

    @Test
    void testGetCallBackResponse() {

        when(walletConfig.getSecretKey()).thenReturn("SECRET_KEY");

        String response =
                mobikwikWalletPaymentDao.getCallBackResponse("MID123", "ORD123");

        verify(paymentDao).saveRequestLog(
                anyString(),
                anyString(),
                anyString(),
                anyString());

        assertEquals(response.contains("ORD123"), true);
    }

    @Test
    void testProcessWalletDoubleVerRequest_SuccessCase() {

        dvResponseDto.setStatuscode(
                PaymentConstants.WALLET_STATUS_SUCCESS_CONST);

        MobikwikWalletDvResponseDto response =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(
                        webResponseDto,
                        dvResponseDto,
                        "DECRYPT_RESPONSE");

        verify(paymentDao).saveResponseLog(
                anyString(),
                anyString(),
                anyString(),
                anyString(),
                anyString());

        verify(statusUpdatePaymentDao)
                .paymentSuccessPendingStatusUpdateGen(
                        anyString(),
                        anyString(),
                        anyString(),
                        anyString());

        assertEquals(
                PaymentConstants.WALLET_STATUS_SUCCESS_CONST,
                response.getStatuscode());
    }

    @Test
    void testProcessWalletDoubleVerRequest_FailureCase() {

        dvResponseDto.setStatuscode(
                PaymentConstants.FAILURE_CONST);

        MobikwikWalletDvResponseDto response =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(
                        webResponseDto,
                        dvResponseDto,
                        "DECRYPT_RESPONSE");

        verify(statusUpdatePaymentDao)
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        anyString());

        assertEquals(
                PaymentConstants.FAILURE_CONST,
                response.getStatuscode());
    }

    @Test
    void testProcessWalletDoubleVerRequest_DefaultCase() {

        dvResponseDto.setStatuscode("UNKNOWN");

        MobikwikWalletDvResponseDto response =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(
                        webResponseDto,
                        dvResponseDto,
                        "DECRYPT_RESPONSE");

        verify(statusUpdatePaymentDao)
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        anyString());

        assertEquals("UNKNOWN", response.getStatuscode());
    }
}



import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.text.MessageFormat;
import java.util.Optional;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class MerchantPricingDaoTest {

    @Mock
    private MerchantOrderHybridFeeRepository merchantOrderHybridFeeRepository;

    @Mock
    private TransactionMapper transactionMapper;

    @InjectMocks
    private MerchantPricingDao merchantPricingDao;

    @Test
    void testGetTransactionAckReq_Success() {

        String atrn = "ATRN123";

        MerchantOrderHybridFee merchantOrderHybridFee = new MerchantOrderHybridFee();
        MerchantPricingDto merchantPricingDto = new MerchantPricingDto();

        when(merchantOrderHybridFeeRepository.findByAtrnNum(atrn))
                .thenReturn(Optional.of(merchantOrderHybridFee));

        when(transactionMapper.mapToMerchantPricingDto(merchantOrderHybridFee))
                .thenReturn(merchantPricingDto);

        MerchantPricingDto response = merchantPricingDao.getTransactionAckReq(atrn);

        assertEquals(merchantPricingDto, response);

        verify(merchantOrderHybridFeeRepository)
                .findByAtrnNum(atrn);

        verify(transactionMapper)
                .mapToMerchantPricingDto(merchantOrderHybridFee);
    }

    @Test
    void testGetTransactionAckReq_ATRNNotFound() {

        String atrn = "INVALID_ATRN";

        when(merchantOrderHybridFeeRepository.findByAtrnNum(atrn))
                .thenReturn(Optional.empty());

        PaymentException exception = assertThrows(
                PaymentException.class,
                () -> merchantPricingDao.getTransactionAckReq(atrn));

        assertEquals(
                ErrorConstants.INVALID_ERROR_CODE,
                exception.getErrorCode());

        assertEquals(
                MessageFormat.format(
                        ErrorConstants.INVALID_ERROR_MESSAGE,
                        PaymentConstants.atrn,
                        PaymentConstants.ATRN_NOT_FOUND),
                exception.getMessage());
    }
}
@Component
@RequiredArgsConstructor
public class MerchantPricingDao {

    private final MerchantOrderHybridFeeRepository merchantOrderHybridFeeRepository;
    private final TransactionMapper transactionMapper;

    public MerchantPricingDto getTransactionAckReq(String atrn) {

        MerchantOrderHybridFee merchantOrderHybridFee = merchantOrderHybridFeeRepository.findByAtrnNum(atrn)
                .orElseThrow(() -> new PaymentException(ErrorConstants.INVALID_ERROR_CODE, MessageFormat
                        .format(ErrorConstants.INVALID_ERROR_MESSAGE, PaymentConstants.atrn, PaymentConstants.ATRN_NOT_FOUND)));
        return transactionMapper.mapToMerchantPricingDto(merchantOrderHybridFee);
    }
}


