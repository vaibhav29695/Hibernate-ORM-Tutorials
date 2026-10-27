external.api.services.endpoint.chargeback-mis=http://localhost:8080

///
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.stereotype.Component;

import com.fasterxml.jackson.databind.ObjectMapper;

import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class ChargebackMisClient extends ApiClient {

    private static final String CHARGEBACK_MIS_DOWNLOAD =
            "/mis/download";

    public ChargebackMisClient(
            String baseUrl,
            String corsOrigin,
            ObjectMapper objectMapper) {

        super(baseUrl, corsOrigin, objectMapper);
    }

    public ReportingResponse<String> downloadMisReport(
            ChargebackMisRequestDto request) {

        log.info("Calling Chargeback MIS download API");

        return post(
                CHARGEBACK_MIS_DOWNLOAD,
                request,
                new ParameterizedTypeReference<ReportingResponse<String>>() {}
        );
    }
}

//////
protected <T, R> ReportingResponse<R> post(
        String urlPath,
        T requestBody,
        ParameterizedTypeReference<ReportingResponse<R>> typeReference) {

    try {
        URI uri = URI.create(getBaseUrl() + urlPath);

        logger.info("Calling URI: {}", uri);
        logger.debug("Request body: {}", requestBody);

        ReportingResponse<R> response = getWebClient()
                .post()
                .uri(uri)
                .headers(headers ->
                        headers.addAll(prepareHttpHeaders()))
                .bodyValue(requestBody)
                .exchangeToMono(clientResponse ->
                        clientResponse.bodyToMono(typeReference))
                .block();

        validateResponse(response);

        return response;

    } catch (NestedRuntimeException e) {
        logger.error(
                "Error while calling API: {}. Exception: {}",
                getBaseUrl() + urlPath,
                e.getMessage());

        throw new ReportingException(
                EXTERNAL_SERVICE_ERROR_CODE,
                MessageFormat.format(
                        EXTERNAL_SERVICE_ERROR_MESSAGE,
                        getServiceName()));
    }
}

/////


import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class ChargebackMisClient extends ApiClient {

    private static final String CHARGEBACK_MIS_DOWNLOAD =
            "/mis/download";

    public ChargebackMisClient(
            String baseUrl,
            String corsOrigin,
            ObjectMapper objectMapper) {

        super(baseUrl, corsOrigin, objectMapper);
    }

    public ReportingResponse<String> downloadMisReport(
            ChargebackMisRequestDto request) {

        log.info("Calling Chargeback MIS API with request: {}", request);

        ReportingResponse<String> response = post(
                CHARGEBACK_MIS_DOWNLOAD,
                request,
                new ParameterizedTypeReference<ReportingResponse<String>>() {}
        );

        log.info("Chargeback MIS API response status: {}",
                response.getStatus());

        return response;
    }
}


///////
import java.time.Duration;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.reactive.function.client.WebClientResponseException;

import lombok.RequiredArgsConstructor;

@Component
@RequiredArgsConstructor
public class ChargebackMisClient {

    private final WebClient.Builder webClientBuilder;

    @Value("${external.api.services.endpoint.chargeback-mis}")
    private String chargebackMisUrl;

    public byte[] downloadMisReport(
            ChargebackMisRequestDto request) {

        try {
            byte[] excelData = webClientBuilder.build()
                    .post()
                    .uri(chargebackMisUrl)
                    .contentType(MediaType.APPLICATION_JSON)
                    .accept(
                        MediaType.parseMediaType(
                            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"),
                        MediaType.APPLICATION_OCTET_STREAM)
                    .bodyValue(request)
                    .retrieve()
                    .bodyToMono(byte[].class)
                    .block(Duration.ofSeconds(60));

            if (excelData == null || excelData.length == 0) {
                throw new IllegalStateException(
                        "Chargeback MIS API returned an empty file");
            }

            return excelData;

        } catch (WebClientResponseException ex) {
            throw new IllegalStateException(
                    "Chargeback MIS API failed. HTTP status: "
                            + ex.getStatusCode(), ex);
        }
    }
}


////

import java.time.LocalDate;

import org.springframework.stereotype.Service;

import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class ReportServiceImpl implements ReportService {

    private final ChargebackMisClient chargebackMisClient;

    @Override
    public byte[] downloadChargebackMis(
            ChargebackMisRequestDto request) {

        if (request == null
                || request.getFromDate() == null
                || request.getToDate() == null) {
            throw new IllegalArgumentException(
                    "From date and To date are required");
        }

        LocalDate fromDate = request.getFromDate();
        LocalDate toDate = request.getToDate();

        if (fromDate.isAfter(toDate)) {
            throw new IllegalArgumentException(
                    "From date cannot be after To date");
        }

        if (request.getChargebackStatus() == null
                || request.getChargebackStatus().isBlank()) {
            request.setChargebackStatus("All");
        }

        return chargebackMisClient.downloadMisReport(request);
    }
}

///////
import java.time.LocalDate;

import lombok.Data;

@Data
public class ChargebackMisRequestDto {

    private LocalDate fromDate;

    private LocalDate toDate;

    private String chargebackStatus = "All";
}

///////

import java.io.IOException;

import javax.servlet.http.HttpServletResponse;

import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

import lombok.RequiredArgsConstructor;

@RestController
@RequestMapping("/reports")
@RequiredArgsConstructor
public class ReportController {

    private final ReportService reportService;

    @PostMapping("/chargeback-mis/download")
    public void downloadChargebackMis(
            @RequestBody ChargebackMisRequestDto request,
            HttpServletResponse response) throws IOException {

        byte[] excelData =
                reportService.downloadChargebackMis(request);

        response.setContentType(
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");

        response.setHeader(
                HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"Chargeback_MIS_Report.xlsx\"");

        response.setContentLength(excelData.length);

        response.getOutputStream().write(excelData);
        response.getOutputStream().flush();
    }
}


////////


import java.io.IOException;

import javax.servlet.http.HttpServletResponse;

import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;

@Component
public class ExcelGenerator {

    public void downloadExcelFile(
            HttpServletResponse response,
            String fileName,
            byte[] excelData) throws IOException {

        response.setContentType(
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");

        response.setHeader(
                HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + fileName + "\"");

        response.setContentLength(excelData.length);

        response.getOutputStream().write(excelData);
        response.getOutputStream().flush();
    }
}

//////
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.math.BigDecimal;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDateTime;
import java.util.List;

import org.apache.poi.ss.usermodel.*;
import org.apache.poi.ss.util.CellRangeAddress;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class ChargebackMisServiceImpl implements ChargebackMisService {

    private final ChargebackBookingRepository chargebackBookingRepository;

    @Override
    @Transactional(readOnly = true)
    public byte[] downloadMisReport(
            ChargebackMisRequestDto request) throws IOException {

        if (request.getFromDate() == null
                || request.getToDate() == null) {
            throw new IllegalArgumentException(
                    "From date and To date are required");
        }

        if (request.getFromDate().isAfter(request.getToDate())) {
            throw new IllegalArgumentException(
                    "From date cannot be after To date");
        }

        String status = request.getChargebackStatus();

        if (status == null || status.isBlank()) {
            status = "All";
        }

        LocalDateTime fromDate =
                request.getFromDate().atStartOfDay();

        LocalDateTime toDateExclusive =
                request.getToDate().plusDays(1).atStartOfDay();

        List<ChargebackBookingEntity> records =
                chargebackBookingRepository.findChargebackMisData(
                        fromDate, toDateExclusive, status);

        String[] headers = {
            "DISPUTE",
            "ATRN",
            "ARN",
            "MID",
            "Merchant Name",
            "Merchant Order No",
            "Reason Codes",
            "REASON As per...",
            "Trnx. Date",
            "Chrg Rec Date",
            "CB TAT",
            "Gateway trace number",
            "Trnx's Amt",
            "Order Amt",
            "REMARKS",
            "STATUS",
            "CARD NUMBER",
            "EPAY TAT",
            "CB send to ME date",
            "Date of Revert Expected",
            "CIN No",
            "Payprocid",
            "TRANSACTION SETTLEMENT",
            "CB ADJUSTED",
            "Additional Remarks"
        };

        // Store Excel in memory and also save it to disk.
        try (Workbook workbook = new XSSFWorkbook();
             ByteArrayOutputStream output =
                     new ByteArrayOutputStream()) {

            Sheet sheet = workbook.createSheet("Chargeback MIS");

            // Header styling
            CellStyle headerStyle = workbook.createCellStyle();

            Font font = workbook.createFont();
            font.setBold(true);
            headerStyle.setFont(font);
            headerStyle.setFillForegroundColor(
                    IndexedColors.GREY_25_PERCENT.getIndex());
            headerStyle.setFillPattern(
                    FillPatternType.SOLID_FOREGROUND);
            headerStyle.setBorderBottom(BorderStyle.THIN);

            Row header = sheet.createRow(0);

            for (int i = 0; i < headers.length; i++) {
                Cell cell = header.createCell(i);
                cell.setCellValue(headers[i]);
                cell.setCellStyle(headerStyle);
            }

            // Populate Excel rows using entity properties
            int rowIndex = 1;

            for (ChargebackBookingEntity record : records) {

                Row row = sheet.createRow(rowIndex++);

                setText(row, 0, record.getCbInstructionType());
                setText(row, 1, record.getAtrnNum());
                setText(row, 2, record.getArnNum());
                setText(row, 3, record.getMerchantId());
                setText(row, 4, record.getMerchantName());
                setText(row, 5, record.getOrderRefNumber());
                setText(row, 6, record.getReasonCode());
                setText(row, 7, record.getReasonDescription());

                setText(row, 8, record.getTransactionDate());
                setText(row, 9, record.getCbRaiseDate());
                setText(row, 10, record.getCbTat());

                setText(row, 11, record.getBankReferenceNumber());

                setAmount(row, 12, record.getGatewayPostingAmount());
                setAmount(row, 13, record.getOrderAmount());

                setText(row, 14, record.getRemarks());
                setText(row, 15, record.getChargebackStatus());
                setText(row, 16, record.getCardNumber());
                setText(row, 17, record.getEpayTat());

                setText(row, 18, record.getChbkDate());
                setText(row, 19, record.getRevertdateExpMerch());
                setText(row, 20, record.getCin());
                setText(row, 21, record.getPayProcId());

                // These two columns require confirmation of the
                // exact source fields used in your existing MIS.
                setText(row, 22, "");
                setText(row, 23, "");
                setText(row, 24, "");
            }

            // Filter, freeze header, and set readable widths.
            sheet.setAutoFilter(new CellRangeAddress(
                    0, Math.max(0, rowIndex - 1),
                    0, headers.length - 1));

            sheet.createFreezePane(0, 1);

            for (int i = 0; i < headers.length; i++) {
                sheet.setColumnWidth(i, 20 * 256);
            }

            // Generate Excel bytes
            workbook.write(output);
            byte[] excelData = output.toByteArray();

            // Save file in the configured folder
            Path folder = Paths.get("C:/ChargebackMIS");
            Files.createDirectories(folder);

            String fileName = "CHARGEBACK_MIS_"
                    + System.currentTimeMillis() + ".xlsx";

            Path filePath = folder.resolve(fileName);

            Files.write(filePath, excelData);

            return excelData;
        }
    }

    private static void setText(
            Row row, int column, Object value) {

        Cell cell = row.createCell(column);

        if (value != null) {
            cell.setCellValue(value.toString());
        } else {
            cell.setCellValue("");
        }
    }

    private static void setAmount(
            Row row, int column, BigDecimal amount) {

        Cell cell = row.createCell(column);

        if (amount != null) {
            cell.setCellValue(amount.doubleValue());
        } else {
            cell.setCellValue(0);
        }
    }
}


/////

// Generate Excel in memory
workbook.write(output);
byte[] excelData = output.toByteArray();

// Folder where MIS Excel files will be stored
Path folderPath = Paths.get("C:/ChargebackMIS");

// Create folder if it does not exist
Files.createDirectories(folderPath);

// Generate unique file name
String fileName = "Chargeback_MIS_Report_"
        + System.currentTimeMillis() + ".xlsx";

Path filePath = folderPath.resolve(fileName);

// Save Excel file into the folder
Files.write(
        filePath,
        excelData,
        StandardOpenOption.CREATE_NEW,
        StandardOpenOption.WRITE
);

// Return Excel data if your API also downloads the file
return excelData;


/////


import jakarta.validation.constraints.NotNull;
import java.time.LocalDate;
import lombok.Data;

@Data
public class ChargebackMisRequestDto {

    @NotNull(message = "From date is required")
    private LocalDate fromDate;

    @NotNull(message = "To date is required")
    private LocalDate toDate;

    private String chargebackStatus = "All";
}


/////
import java.time.LocalDateTime;
import java.util.List;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface ChargebackRepository
        extends JpaRepository<ChargebackEntity, Long> {

    @Query("""
        SELECT c
        FROM ChargebackEntity c
        WHERE c.transactionDate >= :fromDate
          AND c.transactionDate < :toDate
          AND (
              :status = 'All'
              OR c.chargebackStatus = :status
          )
        ORDER BY c.transactionDate DESC
        """)
    List<ChargebackEntity> findChargebackMisData(
            @Param("fromDate") LocalDateTime fromDate,
            @Param("toDate") LocalDateTime toDate,
            @Param("status") String status);
}

/////
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.time.LocalDateTime;
import java.util.List;

import org.apache.poi.ss.usermodel.Row;
import org.apache.poi.ss.usermodel.Sheet;
import org.apache.poi.ss.usermodel.Workbook;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class ChargebackMisServiceImpl
        implements ChargebackMisService {

    private final ChargebackRepository chargebackRepository;

    @Override
    @Transactional(readOnly = true)
    public byte[] downloadMisReport(
            ChargebackMisRequestDto request) throws IOException {

        if (request.getFromDate().isAfter(request.getToDate())) {
            throw new IllegalArgumentException(
                    "From date cannot be after To date");
        }

        String status = request.getChargebackStatus();

        if (status == null || status.isBlank()) {
            status = "All";
        }

        LocalDateTime fromDate =
                request.getFromDate().atStartOfDay();

        LocalDateTime toDateExclusive =
                request.getToDate().plusDays(1).atStartOfDay();

        List<ChargebackEntity> records =
                chargebackRepository.findChargebackMisData(
                        fromDate, toDateExclusive, status);

        try (Workbook workbook = new XSSFWorkbook();
             ByteArrayOutputStream output =
                     new ByteArrayOutputStream()) {

            Sheet sheet = workbook.createSheet("Chargeback MIS");

            // Excel header
            Row header = sheet.createRow(0);
            header.createCell(0).setCellValue("Transaction Date");
            header.createCell(1).setCellValue("Transaction ID");
            header.createCell(2).setCellValue("Chargeback Status");
            header.createCell(3).setCellValue("Amount");

            // Excel data
            int rowIndex = 1;

            for (ChargebackEntity record : records) {
                Row row = sheet.createRow(rowIndex++);

                // Replace these properties with your entity fields.
                row.createCell(0).setCellValue(
                        record.getTransactionDate() == null
                                ? ""
                                : record.getTransactionDate().toString());

                row.createCell(1).setCellValue(
                        record.getTransactionId() == null
                                ? ""
                                : record.getTransactionId());

                row.createCell(2).setCellValue(
                        record.getChargebackStatus() == null
                                ? ""
                                : record.getChargebackStatus());

                if (record.getAmount() != null) {
                    row.createCell(3).setCellValue(
                            record.getAmount().doubleValue());
                } else {
                    row.createCell(3).setCellValue(0);
                }
            }

            for (int i = 0; i < 4; i++) {
                sheet.autoSizeColumn(i);
            }

            workbook.write(output);
            return output.toByteArray();
        }
    }
}

///////
import java.io.IOException;

import javax.servlet.http.HttpServletResponse;
import javax.validation.Valid;

import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

import lombok.RequiredArgsConstructor;

@RestController
@RequestMapping("/api/chargeback/mis")
@RequiredArgsConstructor
public class ChargebackMisController {

    private final ChargebackMisService chargebackMisService;

    @PostMapping("/download")
    public void downloadMisReport(
            @Valid @RequestBody ChargebackMisRequestDto request,
            HttpServletResponse response) throws IOException {

        byte[] excelData =
                chargebackMisService.downloadMisReport(request);

        response.setContentType(
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");

        response.setHeader(
                HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=Chargeback_MIS_Report.xlsx");

        response.setContentLength(excelData.length);

        response.getOutputStream().write(excelData);
        response.getOutputStream().flush();
    }
}
/////////// MIS report 

public class ChargebackController {    //this is controller

    private final ChargebackDashboardService chargebackService;
    private final LoggerUtility logger = LoggerFactoryUtility.getLogger(this.getClass());

    /**
     * Method name : createChargeback
     * Description : Create chargeback request
     *
     * @param  : ChargebackRequest
     * @return object of TransactionResponse
     */
    @PostMapping(value = "/search")
    @Operation(summary = "API for creating chargeback request")
    public TransactionResponse<Object> createChargeback(@RequestBody ChargebackDashboardRequest chargebackDashboardRequest, @RequestParam(defaultValue = "0") int page, @RequestParam(defaultValue = "50") int size) {
        logger.info("Request received to create chargeback : {}", chargebackDashboardRequest);
        return TransactionResponse.builder().data(List.of(chargebackService.getChargebackDetails(chargebackDashboardRequest))).build();
    }
	------------------------
	public class ChargebackDashboardService { //this is service 

    private final ChargebackDao chargebackDao;
    private final ChargebackValidator chargeBackValidator;
    private final LoggerUtility logger = LoggerFactoryUtility.getLogger(this.getClass());

    public List<ChargebackBookingDto> getChargebackDetails(ChargebackDashboardRequest chargebackDashboardRequest) {
        logger.info("Fetching chargeback details");
        chargeBackValidator.validateChargebackDashboardRequest(chargebackDashboardRequest);
        Long fromDate = chargebackDashboardRequest.getFromDate();
        Long toDate = chargebackDashboardRequest.getToDate();
        String mId= chargebackDashboardRequest.getMerchantId();
        String cbStatus= chargebackDashboardRequest.getChargebackStatus();
        String atrn=chargebackDashboardRequest.getAtrn();
        String cbrn=chargebackDashboardRequest.getCbrn();
        logger.info("Fetching chargeback details between fromDate: {} and toDate: {}", fromDate, toDate, mId, cbStatus, atrn, cbrn);
        return chargebackDao.getChargebackDetailsBetweenDate(fromDate, toDate, mId, cbStatus, atrn, cbrn);
    }
}
-------------------------------
public class ChargebackDao { //this is dao
    private final LoggerUtility logger= LoggerFactoryUtility.getLogger(this.getClass());

    private final ChargebackBookingRepository chargebackBookingRepository;
    private final ChargebackBookingMapper chargebackBookingMapper;

    public List<ChargebackBookingDto> getChargebackDetailsBetweenDate(Long updatedDate, Long createdDate,String merchantId, String chargebackStatus, String atrn, String cbrn) {
        logger.info("Fetching chargeback details fromDate={}, toDate={},"+ updatedDate, createdDate);
        List<ChargebackBooking> entityList= chargebackBookingRepository.findChargebackDetails(updatedDate, createdDate, merchantId, chargebackStatus, atrn, cbrn);
        return chargebackBookingMapper.toDtoList(entityList);

    }
	-------------------
package com.epay.transaction.repository;

import com.epay.transaction.entity.ChargebackBooking;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface ChargebackBookingRepository extends JpaRepository<ChargebackBooking, String> {

    @Query(value = """
    SELECT *
    FROM CHARGEBACK_BOOKING WHERE ((:merchantId IS NOT NULL AND CREATED_DATE BETWEEN :fromDate AND :uptoDate) OR (:atrnNum IS NOT NULL AND ATRN_NUM = :atrnNum)
            OR (:cbrnNum IS NOT NULL AND CBRN_NUM = :cbrnNum)) AND (:merchantId IS NULL OR MERCHANT_ID = :merchantId)
        AND (:chargebackStatus IS NULL OR CHARGEBACK_STATUS = :chargebackStatus) ORDER BY CREATED_DATE DESC""", nativeQuery = true)
    List<ChargebackBooking> findChargebackDetails(@Param("fromDate") Long fromDate, @Param("uptoDate")
    Long uptoDate, @Param("merchantId") String merchantId, @Param("chargebackStatus") String chargebackStatus, @Param("atrnNum") String atrnNum, @Param("cbrnNum") String cbrnNum);

}
--------------------------
package com.epay.transaction.mapper;  // this is mapper 

import com.epay.transaction.dto.ChargebackBookingDto;
import com.epay.transaction.entity.ChargebackBooking;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.ReportingPolicy;

import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Date;
import java.util.List;

@Mapper(componentModel = "spring", unmappedSourcePolicy = ReportingPolicy.IGNORE, unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface ChargebackBookingMapper {
//
//
//    ChargebackBookingDto toDto(ChargebackBooking entity);
//    ChargebackBooking toEntity(ChargebackBookingDto dto);
//    List<ChargebackBookingDto> toDtoList(List<ChargebackBooking> entityList);
//    List<ChargebackBooking> toEntityList(List<ChargebackBookingDto> dtoList);


    // Entity to DTO
    @Mapping(source = "cbrnNum", target = "cbrn")
    @Mapping(source = "atrnNum", target = "atrn")
    @Mapping(source = "cin", target = "cinNumber")
    @Mapping(source = "orderRefNumber", target = "merchantOrderNo")
    @Mapping(source = "cbStageCode", target = "cbStage")
    @Mapping(source = "reasonDescription", target = "cbReason")
    @Mapping(source = "cbBookAmount", target = "cbBook")
    @Mapping(source = "orderAmount", target = "merchantOrderAmount")
    @Mapping(source = "cbRaiseDate", target = "cbRaisedDate")
    @Mapping(source = "cbTat", target = "cbTat")
    @Mapping(source = "epayTat", target = "tat")
    @Mapping(source = "remainingMerchantTat", target = "merchantCbTatRemain")
    @Mapping(source = "revertdtExpMerch", target = "merchantRevertExpected")
    ChargebackBookingDto toDto(ChargebackBooking entity);

    // DTO to Entity
    @Mapping(source = "cbrn", target = "cbrnNum")
    @Mapping(source = "atrn", target = "atrnNum")
    @Mapping(source = "cinNumber", target = "cin")
    @Mapping(source = "merchantOrderNo", target = "orderRefNumber")
    @Mapping(source = "cbStage", target = "cbStageCode")
    @Mapping(source = "cbReason", target = "reasonDescription")
    @Mapping(source = "cbBook", target = "cbBookAmount")
    @Mapping(source = "merchantOrderAmount", target = "orderAmount")
    @Mapping(source = "cbRaisedDate", target = "cbRaiseDate")
    @Mapping(source = "cbTat", target = "cbTat")
    @Mapping(source = "tat", target = "epayTat")
    @Mapping(source = "merchantCbTatRemain", target = "remainingMerchantTat")
    @Mapping(source = "merchantRevertExpected", target = "revertdtExpMerch")
    ChargebackBooking toEntity(ChargebackBookingDto dto);

    // Entity list to DTO list
    List<ChargebackBookingDto> toDtoList(List<ChargebackBooking> entityList);
    // DTO list to Entity list
    List<ChargebackBooking> toEntityList(List<ChargebackBookingDto> dtoList);
    // Date -> LocalDate
    default LocalDate map(Date date) {
        if (date == null) {
            return null;
        }
        return date.toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
    }
    // LocalDate -> Date
    default Date map(LocalDate date) {
        if (date == null) {
            return null;
        }
        return Date.from(date.atStartOfDay(ZoneId.systemDefault()).toInstant());

    }
}
--------------------------
package com.epay.transaction.dto;  // this is dto i want this is response

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.util.Date;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class ChargebackBookingDto {

    private String cbrn;
    private String atrn;
    private String pod;
    private String action;
    private String cinNumber;
    private String merchantOrderNo;
    private String cbStage;
    private String cbReason;
    private String merchantId;
    private String merchantName;
    private BigDecimal cbAmount;
    private BigDecimal cbBook;
    private BigDecimal merchantOrderAmount;
    private BigDecimal gatewayPostingAmount;
    private Date transactionDate;
    private Date cbRaisedDate;
    private Date cbTat;
    private Integer tat;
    private Integer merchantCbTatRemain;
    private Date merchantRevertExpected;
    private String chargebackStatus;
}
---------------------------------------------------
public class ChargebackDashboardRequest { // this is my request
    private Long fromDate;
    private Long toDate;
    private String chargebackStatus;
    private String merchantId;
    private String cbrn;
    private String atrn;
}
----------------------------------------------------
@Component
@RequiredArgsConstructor   //this is validator
public class ChargebackValidator extends BaseValidator {

    private static final String TO_DATE = "yyyy-mm-dd";
    private static final String CHARGEBACK_STATUS = "Chargeback";
    private static final String CBRN="12345";
    private final LoggerUtility logger = LoggerFactoryUtility.getLogger(this.getClass());

    /**
     * Validate chargeback dashboard search request.
     *
     * Date rules:
     * 1. ATRN or CBRN provided: dates are optional.
     * 2. Neither ATRN nor CBRN provided: both dates are mandatory.
     * 3. MID and chargebackStatus are optional.
     */
    public void validateChargebackDashboardRequest(ChargebackDashboardRequest request) {
        logger.info("Inside validateChargebackDashboardRequest");
        errorDtoList = new ArrayList<>();
        validateMandatoryFields(request);
        validateLeadingTrailingSpaces(request);
        validateFieldsValue(request);
        if (!hasAtrnOrCbrn(request)) {
            validateChargebackDates(request.getFromDate(), request.getToDate());
        }
        throwIfErrors();
        logger.info("Chargeback search request validation completed");
    }

    /**
     * fromDate and toDate are mandatory only when
     * neither ATRN nor CBRN is supplied.
     */
    protected void validateMandatoryFields(ChargebackDashboardRequest request) {
        logger.info("Validating mandatory fields");
        if (!hasAtrnOrCbrn(request)) {
            checkMandatoryDateField(request.getFromDate(), FROM_DATE);
            checkMandatoryDateField(request.getToDate(), TO_DATE);
            throwIfErrors();
        }
    }

    /**
     * Check whether the request contains ATRN or CBRN.
     */
    private boolean hasAtrnOrCbrn(ChargebackDashboardRequest request) {
        return StringUtils.isNotBlank(request.getAtrn()) || StringUtils.isNotBlank(request.getCbrn());
    }

    /**
     * Validate optional search fields when supplied.
     */
    protected void validateFieldsValue(ChargebackDashboardRequest request) {
        logger.info("Validating chargeback search field values");
        if (StringUtils.isNotBlank(request.getMerchantId())) {
            validateFixedFieldLength(request.getMerchantId(), TransactionConstant.MID_MAX_LENGTH, MID);
            throwIfErrors();
            validateFieldWithRegex(request.getMerchantId(), NUMBER_ONLY, MID, INCORRECT_FORMAT);
            throwIfErrors();
        }
        if (StringUtils.isNotBlank(request.getAtrn())) {
            validateFixedFieldLength(request.getAtrn(), TransactionConstant.ATRN_ARRN_LENGTH, ATRN);
            throwIfErrors();

            // Enable this only if ATRN must contain digits exclusively.
            // validateFieldWithRegex(
            //         request.getAtrn(), NUMBER_ONLY,
            //         ATRN, INCORRECT_FORMAT);
            // throwIfErrors();
        }

        // CBRN is optional.
        if (StringUtils.isNotBlank(request.getCbrn())) {
            // Add your existing CBRN length/format validation here,
            // if applicable to your CBRN specification.
        }

        // Chargeback status is optional.
        // When supplied, use it as a repository search filter.
        // Add status validation here only if your existing
        // business rules require checking allowed status values.
    }

    /**
     * Validate leading/trailing spaces for optional fields.
     */
    private void validateLeadingTrailingSpaces(ChargebackDashboardRequest request) {
        logger.info("Checking leading/trailing spaces");
        checkForLeadingTrailingAndSingleSpace(request.getMerchantId(), MID);
        checkForLeadingTrailingAndSingleSpace(request.getChargebackStatus(), CHARGEBACK_STATUS);
        checkForLeadingTrailingAndSingleSpace(request.getAtrn(), ATRN);
        checkForLeadingTrailingAndSingleSpace(request.getCbrn(), CBRN);
        throwIfErrors();
    }

    /**
     * Validate date range only when dates are required.
     */
    private void validateChargebackDates(Long fromDate, Long toDate) {
        logger.info("Validating chargeback dates");
        checkMandatoryDateField(fromDate, FROM_DATE);
        checkMandatoryDateField(toDate, TO_DATE);
        throwIfErrors();
        if (fromDate > DateTimeUtils.endOfDayMillis()) {
            addError(FROM_DATE, INVALID_ERROR_CODE, MessageFormat.format(INVALID_ERROR_MESSAGE, FROM_DATE, FUTURE_DATE_ERROR));
            throwIfErrors();
        }

        if (toDate > DateTimeUtils.endOfDayMillis()) {
            addError(TO_DATE, INVALID_ERROR_CODE, MessageFormat.format(INVALID_ERROR_MESSAGE, TO_DATE, FUTURE_DATE_ERROR));
            throwIfErrors();
        }

        if (fromDate > toDate) {
            addError(FROM_DATE, INVALID_ERROR_CODE, MessageFormat.format(INVALID_ERROR_MESSAGE, FROM_DATE, FROM_TO_DATE_ERROR));
            throwIfErrors();
        }
        validateSixMonthRange(fromDate, toDate);
        logger.info("Chargeback date validation completed");
    }

    /**
     * Validate that the date range does not exceed six months.
     */
    private void validateSixMonthRange(Long fromDate, Long toDate) {
        LocalDate fromLocalDate = Instant.ofEpochMilli(fromDate).atZone(ZoneId.systemDefault()).toLocalDate();
        LocalDate toLocalDate = Instant.ofEpochMilli(toDate).atZone(ZoneId.systemDefault()).toLocalDate();
        LocalDate maximumDate = fromLocalDate.plusMonths(6);

        if (toLocalDate.isAfter(maximumDate)) {
            addError(FROM_DATE, INVALID_ERROR_CODE, MessageFormat.format(INVALID_ERROR_MESSAGE, FROM_DATE, CHARGEBACK_DATE_RANGE_6_MONTHS_ERROR));
            throwIfErrors();
        }
    }
}
--------------------------------	
